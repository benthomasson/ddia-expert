# Topic: Read the POSIX spec on fsync semantics for directories vs files, and how ext4/XFS/btrfs differ in practice

**Date:** 2026-05-29
**Time:** 11:21

Now I have a thorough picture. Here's the explanation.

---

# `fsync` Semantics: POSIX Spec, Directories vs Files, and Filesystem Differences

## The POSIX Contract

POSIX defines two related syscalls with different scope:

| Syscall | What it flushes | When it returns |
|---------|----------------|-----------------|
| `fsync(fd)` | File data **and** all metadata (size, mtime, permissions, allocation maps) | After data is on stable storage |
| `fdatasync(fd)` | File data **and** only metadata needed for data retrieval (file size if changed) | After data is on stable storage |

The critical distinction POSIX makes: **`fsync` operates on a single file descriptor, not across files**. Calling `fsync(wal_fd)` guarantees the WAL is durable. It says *nothing* about any other file. Cross-file ordering requires explicit sequencing of separate `fsync` calls (documented in `entries/2026/05/29/topic-fsync-ordering-guarantees.md`).

## The Directory Fsync Problem

This is the subtlest part of the POSIX spec, and the one this entire codebase misses.

When you create a new file, two independent metadata changes occur:

1. The **file's inode** — size, timestamps, data block pointers (flushed by `fsync(file_fd)`)
2. The **parent directory's entry** — the mapping from filename to inode (flushed by `fsync(dir_fd)`)

These are separate structures. `fsync` on a file descriptor flushes the file's data and metadata, but **not the directory entry pointing to it**. To make a new filename durable, you must also:

```python
dir_fd = os.open(parent_dir, os.O_RDONLY)
os.fsync(dir_fd)
os.close(dir_fd)
```

**No implementation in this codebase does this** — confirmed by grep returning zero matches for directory-level fsync across all 13 `os.fsync` call sites (`entries/2026/05/29/topic-directory-fsync-gap.md`). Every WAL rotation (`wal.py:122`), SSTable creation (`lsm.py:80`), Bitcask segment rotation, and compaction rename is vulnerable.

The same applies to `os.rename()` — POSIX guarantees atomicity (the name change is all-or-nothing) but **not durability**. Without fsyncing the directory after a rename, a crash can revert the rename entirely. Both Bitcask implementations call `os.rename()` during compaction (`hash-index-storage/bitcask.py:297`, `log-structured-hash-table/bitcask.py:301`) without directory fsync (`entries/2026/05/29/topic-rename-durability-guarantees.md`).

## How ext4, XFS, and btrfs Differ in Practice

### ext4

ext4 has three data modes controlled by the `data=` mount option:

| Mode | Behavior | Directory fsync impact |
|------|----------|----------------------|
| `data=ordered` (default) | Data blocks written before metadata journal commits | Provides some implicit ordering — you won't see stale data blocks — but does **not** replace `fsync`. Directory entries are parent-directory metadata, not file metadata, so they follow directory journal timing. Typical crash window: ~5 seconds (default `commit` interval). |
| `data=journal` | Both data and metadata go through the journal | Strongest guarantees, highest overhead. |
| `data=writeback` | No ordering between data and metadata writes | A crash can expose stale or garbage data blocks. Makes missing `fsync` calls even more dangerous. |

ext4 has an `auto_da_alloc` feature (enabled by default since kernel 2.6.30) that partially mitigates the rename-without-directory-fsync problem for **replace-via-rename** patterns — but it does **not** help for new file creation. Pre-4.13 kernels had a bug where `fsync` in `data=ordered` mode could reorder writes, leading to zero-length files on crash (`entries/2026/05/29/topic-fsync-durability-guarantees.md`).

### XFS

XFS is historically the most aggressive about delayed allocation. Key differences:

- Without `fsync`, file data may not exist on disk **at all** — just metadata intent in the log. XFS keeps data in page cache more aggressively than ext4.
- XFS famously had zero-length file bugs on crash before disk barriers were required by the kernel.
- Directory fsync is **strictly required** for new file visibility after crash. Modern XFS is better about barriers, but the spec hasn't changed.
- Metadata journal flush heuristics can create longer crash windows than ext4's fixed 5-second commit interval.

### btrfs

btrfs uses copy-on-write semantics, but this does **not** eliminate the directory fsync requirement:

- A directory fsync is still required to guarantee a new file's existence after crash.
- btrfs's CoW approach means in-place overwrites aren't truly in-place, which can interact with `fdatasync` optimization assumptions differently than ext4/XFS.

### macOS (APFS) — the current development platform

This codebase runs on Darwin 24.4.0, where `os.fsync()` historically did **not** flush the disk's hardware write cache. Only `fcntl(fd, F_FULLFSYNC)` provides that guarantee. Recent macOS versions have improved `fsync` behavior, but `F_FULLFSYNC` remains the only guaranteed-durable call. **This codebase never uses `F_FULLFSYNC`** — none of the 13 `fsync` call sites may actually be durable against power loss on the current platform (`entries/2026/05/29/topic-fsync-ordering-guarantees.md`).

## The `fdatasync` Optimization Opportunity

The codebase uses `os.fsync()` exclusively at all 13 sync sites and never `os.fdatasync()`. For append-only workloads (WAL appends, Bitcask record writes), `fdatasync` still must flush the size change but skips mtime — a modest saving. The real win is for **in-place page overwrites** in the B-tree `PageManager` (`btree.py:105`), where the file size doesn't change and `fdatasync` can skip the metadata write entirely — potentially 2-3x faster on ext4 (`entries/2026/05/29/topic-fdatasync-vs-fsync-tradeoffs.md`).

However, `os.fdatasync()` is not available on macOS/Darwin via Python's `os` module, requiring a platform-aware fallback:

```python
_fdatasync = getattr(os, 'fdatasync', os.fsync)
```

## Summary: The Durability Hierarchy in This Codebase

| Operation | What survives | What's lost |
|-----------|--------------|-------------|
| `write()` alone | Nothing on any crash | Everything |
| `flush()` only | Process crash (kernel survives) | Power loss, kernel panic |
| `flush()` + `fsync()` | Power loss on Linux (ext4/XFS/btrfs) | Possibly power loss on macOS (no `F_FULLFSYNC`) |
| `flush()` + `fsync()` + directory fsync | Power loss + new file visibility | Nothing (if hardware doesn't lie) |
| `flush()` + `F_FULLFSYNC` + directory fsync | Everything including disk write cache | Hardware failure |

The standalone WAL (`write-ahead-log/wal.py:_do_sync`) reaches level 3. The LSM WAL (`lsm.py:26`) is stuck at level 2. No implementation reaches level 4 or 5.

---

## Topics to Explore

- [general] `ext4-mount-options-durability` — How `data=journal` vs `data=ordered` vs `data=writeback` changes crash semantics and whether any mode compensates for missing directory fsync (none fully do)
- [function] `write-ahead-log/wal.py:_do_sync` — The three sync modes (sync/batch/none) and how `fdatasync` would slot in as an optimization; the central dispatch for all durability decisions
- [general] `macos-f-fullfsync-semantics` — How `fcntl(fd, F_FULLFSYNC)` differs from `fsync` on Darwin/APFS, and why the current codebase may have no durable writes on the development platform
- [function] `log-structured-merge-tree/lsm.py:_flush` — The most dangerous path: SSTable creation with no fsync on file or directory, followed by WAL truncation, creating a window where both can be lost
- [general] `atomic-write-via-temp-rename` — The write-to-temp/fsync/rename/fsync-dir pattern used by SQLite, PostgreSQL, and RocksDB to guarantee both content and existence durability — absent from this entire codebase

## Beliefs

- `no-directory-fsync-anywhere` — None of the 13 `os.fsync` calls in the codebase target a directory file descriptor; all fsync calls operate on data file descriptors only, leaving every file creation, rename, and deletion non-durable at the directory level
- `fsync-semantics-vary-by-filesystem` — `os.fsync()` provides different effective guarantees on ext4 (data+metadata durable), XFS (durable but delayed allocation can create longer crash windows), and macOS/APFS (may not flush disk write cache without `F_FULLFSYNC`)
- `fdatasync-safe-for-all-append-paths` — Every append-only file in the codebase (WAL segments, Bitcask data files) writes sequentially with monotonically increasing file size, making `fdatasync` a safe drop-in for `fsync` on those paths; in-place B-tree page overwrites benefit even more since file size doesn't change
- `posix-fsync-is-per-file-not-cross-file` — POSIX `fsync(fd)` provides no ordering guarantees across different file descriptors; the WAL-before-data ordering relied upon by the B-tree (`btree.py:137` then `btree.py:105`) is enforced only by the sequence of separate fsync calls, not by any single call
- `macos-fsync-not-durable` — On the current platform (Darwin 24.4.0/APFS), `os.fsync()` may not flush the disk write cache; true durability requires `fcntl(fd, F_FULLFSYNC)`, which is never used anywhere in the codebase

