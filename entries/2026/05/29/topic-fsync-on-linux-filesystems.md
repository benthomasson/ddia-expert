# Topic: How ext4 `data=ordered` vs `data=writeback` vs XFS metadata journaling changes whether this bug is silent or catastrophic

**Date:** 2026-05-29
**Time:** 12:21

# How Filesystem Journaling Mode Determines Whether the Missing `fsync` Bug Is Silent or Catastrophic

## The Bug in One Sentence

**No implementation in this codebase ever opens a directory file descriptor for `fsync`** — confirmed by a codebase-wide grep that found zero matches for `O_DIRECTORY`, `dirfd`, or `os.open(dir` patterns. All 13 `os.fsync()` calls target regular file descriptors only. This means that after every file creation (WAL rotation, SSTable flush) and every `os.rename()` (Bitcask compaction), the directory entry pointing to that file is not durably persisted.

The consequences range from "harmless" to "total silent data loss" depending entirely on which filesystem and mount options you're running on.

---

## What the Code Does

Three concrete sites demonstrate the pattern:

### 1. WAL Rotation — `write-ahead-log/wal.py:_rotate` (~line 111–122)

```python
def _rotate(self):
    if self._fd:
        self._fd.flush()
        os.fsync(self._fd.fileno())   # old file contents durable ✓
        self._fd.close()
    # ...
    self._current_file = os.path.join(self._dir, f"{next_num:06d}.wal")
    self._fd = open(self._current_file, "ab")  # creates new file
    # ← no os.fsync on the directory — entry may not survive crash
```

After rotation, `_do_sync` (line 122–133) faithfully fsyncs every write to the new file — giving a **false sense of durability**. The data blocks are on the platter, but the directory doesn't know the file exists.

### 2. Bitcask Compaction — `hash-index-storage/bitcask.py:297`

```python
os.rename(self._data_path(old_active_id), ...)  # line 297
# ← no directory fsync — rename is atomic but not durable
```

`os.rename()` is atomic at the VFS level (never both names, never neither) but **atomic ≠ durable**. The rename modifies directory metadata, not file data. Without fsyncing the directory, the rename can be undone by a crash.

### 3. LSM SSTable Flush — `log-structured-merge-tree/lsm.py:80`

```python
with open(path, "wb") as f:
    # ... writes SSTable entries and footer ...
# ← no fsync on file data, no fsync on directory
```

Then the WAL is truncated (`lsm.py:56–60`), destroying the only other copy of that data. If the directory entry for the SSTable is lost, both copies are gone.

---

## How Each Filesystem Handles the Gap

### ext4 `data=ordered` (the Linux default)

**Behavior:** The journal protects **filesystem consistency** — no corruption after a crash. But it does not guarantee **durability** of recent operations. `data=ordered` means the kernel flushes file data to disk before committing the metadata transaction that references it. This prevents the classic "file has size=4096 in the inode but contains garbage" stale-data exposure bug.

**Effect on the directory fsync gap:**
- File *data* is ordered before metadata, so after an fsync of a file's data, the data blocks are on disk.
- But **directory entries are metadata of the parent directory**, not the file. The directory metadata can sit in the journal commit queue for up to 5 seconds (the default `commit` interval).
- ext4 has an `auto_da_alloc` heuristic (enabled since ~2.6.30) that adds an implicit barrier for the pattern `open(O_TRUNC) → write → close → rename` — the classic "safe save." **But this only covers that narrow pattern.** A rename of a data file during compaction (`bitcask.py:297`) is not a truncate-write-rename sequence, so `auto_da_alloc` does not apply.

**Verdict:** The bug is **usually silent** — most crashes happen outside the 5-second commit window, and `auto_da_alloc` papers over some rename patterns. But under heavy I/O pressure or a power failure at an unlucky moment, the rename or new-file creation is lost. The Bitcask compaction is the highest-risk site: you lose the rename, the old file ID scheme is restored, and the in-memory keydir (rebuilt by `_recover` at line 67 via `_find_file_ids` + `os.listdir`) sees stale file IDs. The compacted data is orphaned — data blocks on disk with no directory entry pointing to them.

### ext4 `data=writeback`

**Behavior:** No ordering between data and metadata. The kernel can write metadata to the journal before the corresponding data blocks reach disk. This is faster than `data=ordered` but exposes stale data on crash (an allocated block that hasn't been overwritten yet contains whatever was there before).

**Effect on the directory fsync gap:**
- Everything the `data=ordered` mode leaves vulnerable, `data=writeback` makes *worse*.
- A directory entry for a new file can be committed to the journal **before the file's data blocks are written**. After crash, the file exists (the directory entry survived) but contains garbage — stale data from a previous file that occupied those blocks.
- For renames: the rename metadata can be reordered freely. A crash can result in the file existing under its old name with partial new data, or under its new name with stale data.

**Verdict:** The bug goes from "silent data loss" to **"silent data corruption"**. The recovery code (`_scan_segment`, `_recover`, `_wal_files`) uses `os.listdir` to discover files and trusts that if a file exists, its contents are valid. With `data=writeback`, a file can exist with garbage contents that happen to pass CRC checks on some records (stale data from a previous incarnation of that file). The CRC only covers the payload — not `seq_num` or record length headers — so corrupted framing metadata is accepted silently (`topic-wal-crc-coverage-gap`).

The compound failure: SSTable file exists (directory entry committed) → file contains stale blocks (data not flushed) → LSM WAL already truncated (`lsm.py:56–60`) → recovery finds a corrupt SSTable and a truncated WAL → **both copies of the data are gone or corrupt**.

### XFS

**Behavior:** XFS delays metadata updates aggressively. It uses write-back metadata logging by default and provides **no implicit durability promises for renames or new file creation** without explicit `fsync` on the directory. In older kernels (pre-4.18), XFS could even lose the file *contents* of a newly allocated file if the directory entry wasn't fsynced, because the allocator metadata might not have been committed.

**Effect on the directory fsync gap:**
- Directory fsync is **required** for rename durability — there is no `auto_da_alloc`-like heuristic.
- The metadata journal commit interval can be longer than ext4's depending on log flush heuristics and I/O pressure.
- XFS's allocator groups metadata aggressively, so the window where a new file's directory entry is uncommitted can be larger than on ext4.

**Verdict:** The bug is **reliably catastrophic** under XFS. The WAL rotation scenario (`wal.py:_rotate`) is the clearest example: the old WAL segment is properly fsynced and closed, the new segment is created and receives writes that are individually fsynced, but the new file's directory entry never reaches disk. A power failure loses the entire new WAL segment. All records written after the rotation are gone, and since the old segment was already closed and its sequence number is stale, the recovery code (`_recover_seq_num`) picks up from the old segment's last record — silently dropping everything that came after.

For Bitcask compaction: old segments are `os.remove()`'d (line 288), then the active file is renamed (line 297). Without directory fsync, XFS can persist the *unlinks* (they modify the directory too, but XFS may batch them with the journal) while losing the rename. The result depends on timing — but the worst case is: old segments deleted, rename lost, compacted data orphaned. Recovery via `_find_file_ids()` + `os.listdir()` sees neither the old nor the new file IDs.

### macOS APFS (for comparison)

**Behavior:** Copy-on-write filesystem with atomic container superblock updates. Metadata and data changes are committed together in CoW transactions.

**Verdict:** The bug is **completely hidden**. APFS provides implicit rename and file-creation durability through its transaction model. This is why all the implementations work correctly on macOS development machines but carry latent Linux bugs. As documented in `topic-directory-fsync-semantics.md`: "This is why the bug hides on macOS development machines and only manifests on Linux production deployments."

---

## The Severity Spectrum

| Filesystem | New file creation | Rename | Compound failure | Severity |
|---|---|---|---|---|
| ext4 `data=ordered` | Dir entry lost, data blocks orphaned | Rename reverted, old name restored | WAL truncated + SSTable dir entry lost = data gone | **Moderate** — usually masked by commit interval |
| ext4 `data=writeback` | File may exist with stale/garbage data | Rename can be reordered with data writes | Corrupt file passes CRC on stale records | **High** — silent corruption, not just loss |
| XFS | Dir entry reliably delayed, wider loss window | No implicit rename durability at all | Old segments deleted + rename lost + compaction orphaned | **Critical** — reliable data loss |
| macOS APFS | Implicitly durable | Implicitly durable | N/A | **None** — bug is invisible |

---

## The Fix

One helper, called after every file creation and rename:

```python
def _fsync_directory(dir_path):
    fd = os.open(dir_path, os.O_RDONLY)
    try:
        os.fsync(fd)
    finally:
        os.close(fd)
```

This must be added to: `_rotate` (wal.py), `_maybe_rotate` (bitcask.py), `SSTable.write` (lsm.py), `compact` after every `os.rename` and `os.remove`, and every hint file creation path.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — The full compaction method (lines 194–321) to verify the exact ordering of `os.remove`, `os.rename`, and data writes, and identify which crash points lose data vs. which are recoverable
- [function] `log-structured-merge-tree/lsm.py:_flush` — The memtable-to-SSTable flush path where WAL truncation races ahead of SSTable durability, creating the compound failure where both copies are lost
- [general] `ext4-auto-da-alloc-coverage` — Which rename patterns are covered by ext4's `auto_da_alloc` heuristic and which are not — specifically whether compaction renames and WAL rotation qualify
- [function] `write-ahead-log/wal.py:_do_sync` — How the three sync modes (sync, batch, none) interact with the directory fsync gap — even "sync" mode only syncs the file, not the directory
- [general] `posix-rename-durability-semantics` — The POSIX specification only guarantees rename atomicity, not durability; trace how PostgreSQL, SQLite, and RocksDB each handle the dir-fsync-after-rename pattern

## Beliefs

- `dir-fsync-gap-severity-is-filesystem-dependent` — The missing directory fsync after file creation and rename is silent on APFS, intermittent on ext4 `data=ordered`, and reliably catastrophic on XFS, because each filesystem makes different implicit metadata durability guarantees
- `ext4-auto-da-alloc-does-not-cover-compaction` — ext4's `auto_da_alloc` heuristic only covers the `open(O_TRUNC) → write → rename` pattern; Bitcask compaction's `write-new → delete-old → rename-active` sequence at `bitcask.py:288-297` is not covered
- `data-writeback-escalates-loss-to-corruption` — On ext4 `data=writeback`, the missing directory fsync can cause a file to exist with stale block contents rather than simply not exist, escalating silent data loss to silent data corruption
- `xfs-has-no-implicit-rename-durability` — XFS provides no `auto_da_alloc`-like heuristic and delays metadata aggressively, making directory fsync mandatory for both new file creation and rename durability
- `lsm-compound-failure-loses-both-copies` — The LSM tree truncates the WAL (`lsm.py:56-60`) after `SSTable.write` which has no fsync at all, so a crash can lose both the SSTable (no dir entry) and the WAL (truncated), destroying the only two copies of the data

