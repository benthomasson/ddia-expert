# Topic: Investigate whether any implementation in the repo fsyncs the parent directory after file deletion, which is required for crash-durable unlinking on Linux

**Date:** 2026-05-29
**Time:** 06:40

# No Implementation fsyncs the Parent Directory After File Deletion

## The Problem

On Linux (and POSIX systems generally), `unlink()` / `os.remove()` modifies the **parent directory's metadata** — it removes the directory entry pointing to the inode. The file data and the directory entry live in different on-disk structures. A crash between the `unlink()` syscall and the next directory metadata flush can leave the directory entry intact, meaning the file "reappears" after reboot. The correct sequence for crash-durable deletion is:

```python
os.remove(path)
dir_fd = os.open(os.path.dirname(path), os.O_RDONLY)
os.fsync(dir_fd)
os.close(dir_fd)
```

## What the Codebase Does

### All fsync calls target file descriptors, never directories

Every `os.fsync()` call across the 13 occurrences operates on a **file's own descriptor** — never on a parent directory fd:

- **`write-ahead-log/wal.py`** (lines 115, 128, 133, 184, 209, 261): All fsync calls use `self._fd.fileno()` or `f.fileno()`, which are WAL segment file handles opened with `open(path, "ab")` or `open(path, "wb")`.
- **`b-tree-storage-engine/btree.py`** (lines 105, 113, 137, 144, 171): All use `self._f.fileno()`, the B-tree data file or WAL file handle.
- **`hash-index-storage/bitcask.py`** (line 88): Uses `self.active_file.fileno()`, the active data segment.

### File deletions that lack directory fsync

The `find_os_unlink` results show 17 `os.remove()` / `os.unlink()` calls. The critical ones in production code (not tests) are:

| File | Line | What's deleted | Directory fsync? |
|------|------|---------------|-----------------|
| `log-structured-merge-tree/lsm.py` | 353 | Compacted SSTable files (`sst.path`) | **No** |
| `write-ahead-log/wal.py` | 201 | Truncated WAL segments | **No** |
| `hash-index-storage/bitcask.py` | 288, 290 | Old data/hint segments after compaction | **No** |
| `log-structured-hash-table/bitcask.py` | 292, 295 | Old segments/hints after compaction | **No** |
| `batch-word-count/pipeline.py` | 184 | Intermediate files | **No** |

### The `dir_fd` / `O_DIRECTORY` search confirms the gap

The `find_dir_fd_usage` search returned only two false positives — `open(os.path.join(...))` calls that open regular files within directories, not directory file descriptors. There is **zero** use of `os.open(dir, os.O_RDONLY)` followed by `os.fsync(dir_fd)` anywhere in the codebase.

### Rename operations also skip directory fsync

The two `os.rename()` calls (`hash-index-storage/bitcask.py:297`, `log-structured-hash-table/bitcask.py:301`) modify directory entries in both the source and destination directories. Neither fsyncs either parent directory afterward, so renames are also not crash-durable with respect to directory metadata.

## Practical Impact

The most consequential gap is in **compaction paths**. In `lsm.py:353`, after merging SSTables into a new file, the old SSTable is deleted. If the system crashes before the directory metadata is flushed, the old SSTable reappears on recovery. Depending on the implementation's recovery logic, this could cause:

1. **Duplicate data** — both old and new SSTables are visible
2. **Read amplification** — stale SSTables are scanned unnecessarily
3. **Incorrect merges** — a subsequent compaction might re-merge already-merged data

The WAL truncation path in `wal.py:201` has a similar risk: a truncated segment could reappear, causing replayed operations to be applied twice (though idempotent replay logic may mitigate this).

## Topics to Explore

- [function] `write-ahead-log/wal.py:truncate` — The truncation logic deletes WAL segments without directory fsync; trace whether replay is idempotent enough to tolerate phantom reappearance
- [function] `log-structured-merge-tree/lsm.py:compact` — Read the full compaction method to see whether recovery can handle a crash between SSTable deletion and directory flush
- [function] `hash-index-storage/bitcask.py:compact` — Bitcask compaction deletes old segments and renames the active file — both operations need directory fsync for durability
- [general] `rename-durability-guarantees` — `os.rename()` is atomic on POSIX but the directory entry change is only durable after fsyncing both source and destination directories
- [general] `fsync-on-new-file-creation` — Creating a new file (e.g., new WAL segment in `_rotate()`) also modifies the parent directory; none of the `_rotate` paths fsync the directory after creating the new segment

## Beliefs

- `no-dir-fsync-after-unlink` — No implementation in the repo fsyncs the parent directory after `os.remove()` or `os.unlink()`, making all file deletions non-crash-durable on Linux
- `all-fsyncs-target-file-fds` — Every `os.fsync()` call in the codebase operates on a regular file descriptor, never a directory file descriptor
- `compaction-deletion-not-crash-safe` — SSTable and segment deletions during compaction (in `lsm.py`, `bitcask.py`) can be undone by a crash because directory metadata is not fsynced
- `rename-not-crash-durable` — The `os.rename()` calls in both Bitcask implementations do not fsync source or destination directories, so the rename may not survive a crash

