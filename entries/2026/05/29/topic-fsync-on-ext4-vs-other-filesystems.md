# Topic: ext4 with `auto_da_alloc` partially mitigates the rename-without-directory-fsync problem for replace-via-rename patterns, but not for new file creation. Explore which filesystem behaviors mask or expose these gaps.

**Date:** 2026-05-29
**Time:** 12:48

# ext4 `auto_da_alloc` and Directory Fsync Gaps in This Codebase

## The Core Problem

POSIX requires three steps to durably create or replace a file:
1. Write data to the file
2. `fsync()` the file
3. `fsync()` the **parent directory** to persist the directory entry (the name→inode mapping)

Without step 3, a crash can leave you with a file that was written and synced but whose directory entry never reached disk — the file effectively vanishes. ext4's `auto_da_alloc` mount option (enabled by default since ~2.6.30) partially addresses this, but only for **replace-via-rename** patterns where `rename()` overwrites an existing file. It forces the filesystem to flush the new file's data allocations before completing the rename, preventing the classic "zero-length file after crash" scenario.

Critically, `auto_da_alloc` does **nothing** for newly created files that don't replace an existing entry.

## What This Codebase Does — and Doesn't Do

### No directory fsync anywhere

Across all 13 `os.fsync()` calls in the codebase, **every single one syncs a file descriptor**, never a directory. The canonical pattern for directory fsync in Python would be:

```python
dir_fd = os.open(parent_dir, os.O_RDONLY)
os.fsync(dir_fd)
os.close(dir_fd)
```

This pattern appears zero times. This means the durability of new directory entries depends entirely on filesystem behavior.

### Rename patterns (partially masked by `auto_da_alloc`)

The two `os.rename()` calls are both in Bitcask compaction:

- **`hash-index-storage/bitcask.py:297`** — renames the old active data file during compaction
- **`log-structured-hash-table/bitcask.py:301`** — same pattern

These renames move an existing file to a new name within the same directory. If this is a **replace** (the target path already exists), `auto_da_alloc` on ext4 would force data allocation flush before the rename completes, partially preventing data loss. However, looking at the Bitcask rotation logic (`_maybe_rotate` at line ~101), the active file ID is incremented and a fresh file opened — this is a **new file creation**, not a replace. The subsequent rename during compaction may or may not target an existing path depending on compaction state.

**Bottom line:** `auto_da_alloc` only helps if `os.rename()` overwrites a pre-existing destination. These renames are file-to-new-name moves, so the protection is inconsistent at best.

### New file creation (fully exposed)

Several modules create new files as part of normal operation, with no directory fsync:

| Module | Operation | Where | Exposure |
|--------|-----------|-------|----------|
| WAL | `_rotate()` creates a new `.wal` file | `write-ahead-log/wal.py:120-121` | High — a crash right after rotation could lose the new WAL file entirely |
| B-tree WAL | Opens new WAL file on init | `b-tree-storage-engine/btree.py:128` | Medium — only on first open |
| LSM SSTables | `SSTable.write()` creates new files during flush | `log-structured-merge-tree/lsm.py:80-97` | High — flushed memtable data could vanish |
| Bitcask | `_open_active_file()` creates new data file | `hash-index-storage/bitcask.py:71` | High — every rotation creates a new file |
| Bitcask hints | `_write_hint_file()` creates hint files | `hash-index-storage/bitcask.py:156-163` | Medium — hint loss only costs rebuild time |

`auto_da_alloc` provides **zero protection** for any of these. On ext4 without explicit directory fsync, a power loss after creating and writing to a new file can result in the directory entry never being persisted, even though the file data was `fsync()`'d.

### The LSM WAL: doubly exposed

The LSM tree's WAL (`log-structured-merge-tree/lsm.py:26`) is particularly concerning — it calls `self._fd.flush()` but **never calls `os.fsync()`** at all. This means neither the file data nor the directory entry is guaranteed durable. The WAL's `truncate()` method (line ~56) also reopens the file without any sync. On any filesystem, not just ext4, this WAL provides no crash durability guarantee.

Compare this with `write-ahead-log/wal.py:114-115`, which correctly pairs `flush()` + `fsync()` on every write in sync mode.

## Which Filesystems Mask vs. Expose These Gaps

| Filesystem | Behavior | Effect on this code |
|------------|----------|---------------------|
| **ext4 + `auto_da_alloc`** (default) | Flushes data before replace-via-rename; periodic journal commits flush directory entries | Partially masks rename gaps; new file creation still exposed in the ~5s window between journal commits |
| **ext4 with `data=journal`** | All data and metadata journaled together | Masks most gaps but at significant performance cost |
| **XFS** | No `auto_da_alloc` equivalent; aggressive delayed allocation | Fully exposes both rename and new-file gaps; historically infamous for zero-length files after crash |
| **btrfs** | COW semantics; metadata and data written together on `fsync()` | Partially masks — `fsync()` on a file also persists its directory entry in recent kernels, but this is an implementation detail, not a guarantee |
| **ZFS** | Transactional writes with periodic txg syncs (~5s) | Masks within the txg sync window; data written in the same txg as the directory entry will either both persist or both be lost |

## The Practical Impact

For this codebase, the gaps matter most during:

1. **WAL rotation** (`wal.py:120`) — if the old WAL is closed and the new one created but the directory entry isn't persisted, recovery after crash finds neither the old committed WAL (closed, potentially unlinked after truncation) nor the new one. Data written after rotation is lost.

2. **SSTable flush** (`lsm.py:80-97`) — the memtable is cleared after flush. If the SSTable file's directory entry isn't durable, the memtable data is gone from memory and the SSTable is invisible on disk. The WAL could replay it, except the LSM WAL doesn't fsync either.

3. **Bitcask compaction** (`bitcask.py:297`) — the merge output file could vanish, leaving stale data files already deleted. The combination of deleting old files and creating new ones without directory fsync is the most dangerous pattern.

On ext4 with default mount options, the journal commit interval (typically 5 seconds) provides a probabilistic safety net — most directory entries will be flushed within a few seconds. But "usually durable within 5 seconds" is not the contract a storage engine should rely on.

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:WAL.append` — WAL that only calls flush() without fsync(), making it a no-op for durability on any filesystem
- [function] `hash-index-storage/bitcask.py:compact` — Compaction deletes old files and creates new ones; the full rename + unlink + create sequence without directory fsync is the highest-risk pattern
- [function] `write-ahead-log/wal.py:_rotate` — WAL rotation creates a new file and closes the old one; the window between creation and the next journal commit is the vulnerability
- [general] `directory-fsync-audit` — None of the 13 fsync calls target a directory fd; adding `os.fsync(os.open(dir, O_RDONLY))` after file creation/rename would close all these gaps portably
- [general] `fsync-on-close-semantics` — Python's `open()` in append mode doesn't fsync on close; the btree's `PageManager.close()` at line 113 does this correctly, but the LSM WAL's `close()` does not

## Beliefs

- `no-directory-fsync-anywhere` — No module in this codebase calls `os.fsync()` on a directory file descriptor; all 13 fsync calls target data file descriptors only
- `lsm-wal-has-no-fsync` — The LSM tree's WAL (`lsm.py:26`) calls only `flush()`, never `os.fsync()`, providing no crash durability guarantee on any filesystem
- `wal-rotation-creates-new-files` — WAL rotation (`wal.py:120-121`) creates new numbered `.wal` files rather than replacing existing ones, so ext4 `auto_da_alloc` provides no protection for the new directory entry
- `bitcask-rename-is-not-replace` — The Bitcask `os.rename()` calls move files to new names during compaction rather than atomically replacing existing files, so `auto_da_alloc`'s replace-via-rename protection does not reliably apply
- `flush-fsync-pairing-inconsistent` — `write-ahead-log/wal.py` consistently pairs `flush()` + `fsync()` (lines 114-115, 127-128), while `log-structured-merge-tree/lsm.py` only calls `flush()` (line 26), creating an inconsistent durability contract across modules

