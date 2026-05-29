# Topic: Even with atomic rename, the directory entry must be fsynced for the rename to survive a crash; none of the implementations fsync directories

**Date:** 2026-05-29
**Time:** 08:29

# The Missing Directory Fsync After Rename

## The Problem

On POSIX systems, `os.rename()` is atomic — the kernel guarantees that if a crash happens mid-rename, the file appears under exactly one name, never both, never neither. But **atomic does not mean durable**. The rename modifies a *directory entry*, and that directory entry lives in the directory's metadata, which sits in its own page cache buffer. Without fsyncing the directory's file descriptor, the OS can reorder writes so the new file data reaches disk but the directory entry pointing to it does not. After a power loss, the old name reappears.

The correct sequence is:

```python
os.rename(old_path, new_path)
dirfd = os.open(os.path.dirname(new_path), os.O_RDONLY)
os.fsync(dirfd)
os.close(dirfd)
```

## Where the Gap Exists

The grep for `os.rename` found two call sites:

1. **`hash-index-storage/bitcask.py:297`** — renames the old active data file during compaction
2. **`log-structured-hash-table/bitcask.py:301`** — same pattern, renaming old active file after rotation

The grep for `os.open.*O_DIRECTORY` / `os.opendir` / `dirfd` returned **zero matches** across the entire codebase. No implementation ever opens a directory file descriptor.

## What They Do Fsync

The implementations *do* fsync file data correctly in several places:

- **`hash-index-storage/bitcask.py:88`** — fsyncs the active data file after each write when `sync_writes=True`
- **`write-ahead-log/wal.py:115,128,133,184,209,261`** — fsyncs the WAL file at rotation, after sync-mode writes, during truncation
- **`b-tree-storage-engine/btree.py:105,113,137,144,171`** — fsyncs the page file and WAL after page writes and commits

Every one of these fsyncs a **file descriptor for a regular file** (`self._fd.fileno()`, `self._f.fileno()`). None opens the *containing directory* and fsyncs that.

## Why It Matters

Consider the compaction path in `hash-index-storage/bitcask.py`. During `compact()`, the engine:

1. Writes live entries to new data files
2. Fsyncs the data file contents (line 88, via `_write_record`)
3. Calls `os.rename()` to move old files into their new names (line 297)

If power is lost between steps 2 and the directory metadata reaching disk, the rename is lost. The data file contents are durable (they were fsynced), but the directory still points to the old name. On recovery, the engine sees stale file IDs, and the compacted data is orphaned.

The same risk applies to WAL rotation in `write-ahead-log/wal.py`. The `_rotate()` method (around line 112) creates new WAL files — but the new file's directory entry is never fsynced. A crash could lose the new file entirely, with the WAL appearing to still be the old, pre-rotation file.

## Practical Impact

On ext4 with default mount options (`data=ordered`), metadata updates are often flushed alongside data, so this bug rarely manifests. On XFS, btrfs, or ext4 with `data=writeback`, the window is real. Production databases like SQLite, PostgreSQL, and LevelDB all fsync the directory after renames for exactly this reason.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — Trace the full compaction flow to see every rename and understand what state is lost if the directory entry doesn't survive
- [function] `write-ahead-log/wal.py:_rotate` — WAL rotation creates new files whose directory entries are also unfsynced; understand how this interacts with crash recovery
- [function] `b-tree-storage-engine/btree.py:commit` — The B-tree WAL commit truncates and fsyncs, but never fsyncs the directory; compare with how SQLite's WAL handles this
- [general] `fsync-on-linux-filesystems` — How ext4 `data=ordered` vs `data=writeback` vs XFS metadata journaling changes whether this bug is silent or catastrophic
- [file] `log-structured-hash-table/bitcask.py` — The second Bitcask variant with the same rename-without-dir-fsync pattern; compare its compaction path with the hash-index version

## Beliefs

- `no-dir-fsync-anywhere` — No file in the repository opens a directory file descriptor or calls fsync on a directory; all fsync calls target regular file descriptors
- `rename-without-durability` — Both `hash-index-storage/bitcask.py:297` and `log-structured-hash-table/bitcask.py:301` call `os.rename()` without fsyncing the parent directory, so renames may not survive a power loss
- `file-data-fsync-is-correct` — The WAL, B-tree, and Bitcask implementations all correctly fsync file *contents* after writes, so the durability gap is specifically in directory metadata, not file data
- `wal-rotation-creates-unfsynced-entries` — `write-ahead-log/wal.py:_rotate` creates new `.wal` files whose directory entries are never fsynced, meaning a crash during rotation could lose the new WAL segment entirely

