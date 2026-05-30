# Topic: Investigate whether either implementation provides true durability guarantees, given that the log-structured variant never calls fsync and neither fsyncs the directory

**Date:** 2026-05-29
**Time:** 12:17

# Durability Guarantees Across Storage Implementations

## The Short Answer: Neither Is Fully Durable

The two Bitcask variants sit at opposite ends of the fsync spectrum, but both share the same directory-level gap — and the log-structured variant is significantly worse.

## File-Level Durability

### Hash index storage (`hash-index-storage/bitcask.py`) — fsyncs data, conditionally

This implementation gets file-level durability right when `sync_writes=True` (the default). In `_write_record` at lines 95–96:

```python
self.active_file.flush()
if self.sync_writes:
    os.fsync(self.active_file.fileno())
```

Every `put` and `delete` forces data from OS page cache to stable storage. This is the correct two-step: `flush()` pushes Python's userspace buffer to the kernel, then `os.fsync()` forces the kernel to write through to disk. A crash after `put` returns means the record is on disk.

### Log-structured hash table (`log-structured-hash-table/bitcask.py`) — never fsyncs

In `_write_record` at line 155:

```python
self._active_file.write(header + payload)
self._active_file.flush()
```

That's it. No `os.fsync()` anywhere in this file. `flush()` alone only guarantees data reaches the OS kernel buffer — it does **not** guarantee the data reaches the physical disk. On a power failure, the kernel's dirty page cache is lost. This means:

- A `put()` can return successfully while the data exists only in volatile RAM.
- Crash recovery via `_scan_segment` (line 89) will replay whatever actually made it to disk, silently losing any records that were only in the page cache.
- The CRC check at line 101 will catch partially-written records (torn writes), but it can't recover data that never left RAM.

### LSM Tree WAL (`log-structured-merge-tree/lsm.py`) — same problem

The LSM tree's internal WAL class has the same deficiency. At line 26:

```python
self._fd.flush()
```

No fsync. The entire point of a WAL is to provide durability before acknowledging a write, and this one doesn't. A crash loses any WAL entries still in the page cache, which means the memtable data they were protecting is also lost.

### Standalone WAL (`write-ahead-log/wal.py`) — gets it right

For contrast, the standalone WAL implementation does this correctly. Its `_do_sync` method at lines 124–133 consistently pairs `flush()` with `os.fsync()`, and even supports a `batch` mode that amortizes the fsync cost across multiple writes. This is the implementation that understands the durability contract.

## The Missing Directory Fsync

**Neither implementation fsyncs the parent directory after creating a new file.** This is a subtler but equally critical gap.

When `_rotate_segment` in the log-structured variant creates a new segment file (`log-structured-hash-table/bitcask.py`, line 123), or when the hash-index variant rotates in `_maybe_rotate` (`hash-index-storage/bitcask.py`, line 109), they open a new file — but the directory entry linking that filename to its inode lives in the directory's metadata. Without:

```python
dir_fd = os.open(self._dir, os.O_DIRECTORY)
os.fsync(dir_fd)
os.close(dir_fd)
```

a crash after file creation but before the directory is flushed can leave the file's data on disk but the filename missing from the directory. On recovery, `_find_existing_segments` / `_find_file_ids` would never see it.

The same gap exists for hint files written during compaction (`_write_hint_file` in both implementations). A crash at the wrong moment could leave a compacted data file without its hint file, forcing a slower full scan — or worse, leave a hint file pointing to a data file whose directory entry was lost.

The B-tree's `PageManager` avoids this particular problem by writing to a single pre-existing data file (`b-tree-storage-engine/btree.py`, line 35), so it never needs a directory fsync for normal operations. Its WAL file is also opened at construction time. But even it would need a directory fsync if the WAL file were newly created.

## Practical Impact

| Implementation | File fsync | Directory fsync | Durability on power loss |
|---|---|---|---|
| `hash-index-storage/bitcask.py` | Yes (default) | No | Data durable, new files at risk |
| `log-structured-hash-table/bitcask.py` | **No** | No | **Any recent write can be lost** |
| `log-structured-merge-tree/lsm.py` (WAL) | **No** | No | **WAL defeats its own purpose** |
| `write-ahead-log/wal.py` | Yes | No | Data durable, rotated files at risk |
| `b-tree-storage-engine/btree.py` | Yes (on sync/close) | No | Durable for single-file ops |

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — Compaction creates new data files and hint files without fsyncing either the files or the directory, compounding the durability gap
- [function] `write-ahead-log/wal.py:_do_sync` — The cleanest fsync implementation in the repo; compare its batch mode against the log-structured variant's complete absence of fsync
- [function] `b-tree-storage-engine/btree.py:PageManager.close` — One of the few places that fsyncs before closing; trace whether all crash paths reach this
- [general] `rename-based-atomic-file-creation` — Neither implementation uses the write-to-temp-then-rename pattern for SSTable/segment creation, which is the standard approach to avoid partial files on crash
- [function] `log-structured-merge-tree/lsm.py:WAL.truncate` — Truncates the WAL without fsyncing the truncation, creating a window where a crash could replay already-applied entries

## Beliefs

- `log-structured-no-fsync` — `log-structured-hash-table/bitcask.py:_write_record` calls `flush()` but never `os.fsync()`, so acknowledged writes can be lost on power failure
- `hash-index-conditional-fsync` — `hash-index-storage/bitcask.py:_write_record` fsyncs each write when `sync_writes=True` (the constructor default), providing file-level durability
- `no-directory-fsync-anywhere` — No storage engine in the repo fsyncs the parent directory after creating new segment, SSTable, WAL, or hint files, leaving new file visibility at risk after crash
- `lsm-wal-flush-only` — The LSM tree's internal WAL (`log-structured-merge-tree/lsm.py:WAL.append`, line 26) only calls `flush()`, undermining the crash-recovery guarantee a WAL is supposed to provide
- `standalone-wal-proper-sync` — `write-ahead-log/wal.py:_do_sync` correctly pairs `flush()` with `os.fsync()` in all three sync modes (`sync`, `batch`, `none`), serving as the reference durability implementation

