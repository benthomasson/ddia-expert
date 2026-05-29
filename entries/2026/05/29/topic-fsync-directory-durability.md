# Topic: Even with atomic rename, the directory entry must be fsynced separately on Linux for the rename to survive a power loss; none of the implementations in this repo do this

**Date:** 2026-05-29
**Time:** 08:04

# The Missing Directory Fsync After Rename

## The Problem

On Linux (and POSIX systems generally), `os.rename()` is atomic at the filesystem level — the rename either happens or it doesn't. But **atomic does not mean durable**. After a rename, the new directory entry exists only in the kernel's page cache. If power is lost before the filesystem flushes that directory metadata to disk, the rename vanishes. The old filename reappears as if nothing happened.

The fix is straightforward: after renaming, open the *directory* containing the file and fsync it:

```python
os.rename(old_path, new_path)
dir_fd = os.open(os.path.dirname(new_path), os.O_RDONLY)
os.fsync(dir_fd)
os.close(dir_fd)
```

This forces the directory entry — the mapping from filename to inode — to stable storage.

## Where This Matters in the Repo

Two implementations use `os.rename()`:

1. **`hash-index-storage/bitcask.py:297`** — renames a data file during compaction
2. **`log-structured-hash-table/bitcask.py:301`** — same pattern, renaming the old active file

Both Bitcask implementations rename data files as part of compaction, transitioning them from "active" to "immutable" status. If power is lost after the rename but before the directory is fsynced, the file reverts to its old name. On recovery, the engine won't find the file where it expects it, and depending on the recovery logic, data could be lost or the index could become inconsistent.

## What the Implementations Do Get Right

The codebase is generally careful about fsyncing *file contents*:

- **`write-ahead-log/wal.py`** calls `os.fsync()` in `_do_sync()` (line 128), `_rotate()` (line 115), and `truncate()` (lines 184, 209) — 6 fsync calls total on the WAL file descriptor.
- **`b-tree-storage-engine/btree.py`** fsyncs after every WAL entry (line 137), on WAL commit (line 144), after recovery replay (line 171), and via `PageManager.sync()`/`close()` (lines 105, 113).
- **`hash-index-storage/bitcask.py`** fsyncs the active data file after every write when `sync_writes=True` (line 88).

So the *data* reaches disk. The problem is narrower: the *directory metadata* that says "this file is now named X instead of Y" does not.

## Why This Is Easy to Miss

Most developers think of fsync as "flush my file's data." But a POSIX filesystem tracks two separate things: the file's content (data blocks + inode) and the directory's content (the list of name→inode mappings). Fsyncing the file flushes the former. Fsyncing the *directory* flushes the latter. They are independent operations.

On ext4 with the default `data=ordered` mount option, metadata updates are journaled and *usually* survive crashes even without an explicit directory fsync. This makes the bug nearly impossible to reproduce in normal testing. But "usually" is not "guaranteed" — the POSIX spec does not require it, and other filesystems (XFS, btrfs) have different behavior. Production databases like SQLite, PostgreSQL, and LevelDB all fsync the directory after renames for exactly this reason.

## The Broader Pattern

This gap also applies to file *creation*. When `wal.py` rotates to a new WAL file at line 119 (`self._fd = open(self._current_file, "ab")`), the new file's directory entry isn't fsynced either. The same is true for SSTable creation in `log-structured-merge-tree/lsm.py` and `sstable-and-compaction/sstable.py` — new files are written and closed, but the directory is never fsynced afterward. If power is lost, the file could vanish from the directory even though its data blocks exist on disk (orphaned inode).

None of the 13 `os.fsync()` calls across the repo target a directory file descriptor. Every single one operates on a data file.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — The compaction flow where rename happens; trace what recovery sees if the rename is lost
- [function] `write-ahead-log/wal.py:_rotate` — New WAL file creation without directory fsync; understand the window of vulnerability during rotation
- [general] `posix-durability-guarantees` — Read the POSIX spec on fsync semantics for directories vs files, and how ext4/XFS/btrfs differ in practice
- [file] `b-tree-storage-engine/btree.py` — The WAL + PageManager interaction; this implementation avoids rename entirely by using a truncate-based WAL commit, which sidesteps the directory fsync issue but introduces a different one
- [general] `sqlite-durability-model` — SQLite's approach to directory fsyncs and the `SQLITE_FCNTL_SYNC` mechanism as a reference implementation of correct behavior

## Beliefs

- `no-directory-fsync-anywhere` — All 13 `os.fsync()` calls in the repo target data file descriptors; none open and fsync a directory, leaving rename and file-creation durability to the filesystem's journaling behavior
- `two-rename-sites-in-compaction` — `os.rename()` is used in exactly two files (`hash-index-storage/bitcask.py:297` and `log-structured-hash-table/bitcask.py:301`), both during Bitcask compaction to transition data files from active to immutable
- `wal-rotation-creates-files-without-dir-fsync` — `WriteAheadLog._rotate()` creates new WAL files (`wal.py:119`) but never fsyncs the containing directory, so a crash after rotation could lose the new file's directory entry
- `file-content-fsync-is-thorough` — The WAL, B-tree PageManager, and Bitcask implementations all fsync file *contents* consistently after writes, making the directory fsync gap a subtle rather than gross durability defect

