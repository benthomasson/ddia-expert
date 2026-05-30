# Topic: None of the implementations use the write-to-temp-then-rename pattern for creating new files (SSTables, compacted segments). This is the standard approach for crash-safe file creation and interacts directly with the directory fsync question.

**Date:** 2026-05-29
**Time:** 12:47

# Missing Write-to-Temp-Then-Rename Pattern in Storage Implementations

## The Pattern and Why It Matters

The **write-to-temp-then-rename** pattern is the standard technique for crash-safe file creation in storage engines:

1. Write the new file to a temporary path (e.g., `sstable-007.tmp`)
2. `fsync()` the file to ensure data hits disk
3. `rename()` the temp file to its final path (atomic on POSIX)
4. `fsync()` the parent directory to ensure the directory entry is durable

This guarantees that at any point during a crash, the final filename either doesn't exist (crash before rename) or contains complete, valid data (crash after rename). Without it, a crash mid-write leaves a partially-written file at the final path — an SSTable with a valid name but corrupt contents.

## What the Implementations Actually Do

### SSTable Writer (`sstable-and-compaction/sstable.py:49-97`)

`SSTableWriter.__init__` opens the file directly at its final path:

```python
self._f = open(filepath, "wb")
```

The `finish()` method (line ~85) writes the index, footer, then **seeks back to offset 0** to rewrite the header with the correct entry count. No temp file, no rename, no fsync. A crash during `finish()` leaves a partially-written SSTable at the production path — the sparse index or footer could be truncated, or the header could still contain count 0.

### LSM Tree SSTable Creation (`log-structured-merge-tree/lsm.py:80-99`)

`SSTable.write()` does the same — opens directly at the final path:

```python
with open(path, "wb") as f:
```

It writes entries, then appends the sparse index footer. No temp file, no rename. The `with` block ensures `close()` but not `fsync()`. The WAL (line 26) calls `self._fd.flush()` but never `os.fsync()` — `flush()` only pushes data from Python's buffer to the OS page cache, not to disk.

### Hash Index / Bitcask Variants

The `os.rename` calls found in `hash-index-storage/bitcask.py:297` and `log-structured-hash-table/bitcask.py:301` are for **segment rotation** (renaming the active segment to a frozen segment), not for crash-safe file creation. The rename happens after the file is already populated at its original path — it's a namespace management operation, not a durability operation.

## The Cascade of Missing Durability

The absence of temp-rename connects to a broader pattern:

1. **No `os.fsync()` on data files** — The grep results show `flush()` calls but zero `os.fsync()` calls in the storage modules. `flush()` is Python-to-OS, not OS-to-disk.
2. **No directory `fsync()`** — Even if file contents were synced, the directory entry for a new SSTable isn't synced, so after a crash the file might not appear in the directory listing.
3. **No atomic visibility** — Because files are written directly to their final paths, a reader scanning the data directory could encounter a half-written SSTable file and attempt to open it.

The WAL module (`write-ahead-log/`) does have `sync_mode="sync"` support (test file shows it), suggesting the dedicated WAL implementation understands durability — but the LSM and SSTable modules that *consume* WAL recovery don't apply the same discipline to their own output files.

## What a Correct Implementation Would Look Like

```python
# In SSTableWriter.finish():
self._f.close()
os.fsync(os.open(self._filepath, os.O_RDONLY))  # sync data

tmp_path = self._filepath + ".tmp"
final_path = self._filepath
os.rename(tmp_path, final_path)                  # atomic swap

dir_fd = os.open(os.path.dirname(final_path), os.O_RDONLY)
os.fsync(dir_fd)                                 # sync directory
os.close(dir_fd)
```

This is exactly what production systems like LevelDB, RocksDB, and SQLite do.

## Topics to Explore

- [file] `write-ahead-log/wal.py` — The dedicated WAL module likely implements proper fsync — comparing its durability discipline with the SSTable/LSM modules highlights the gap
- [function] `log-structured-merge-tree/lsm.py:_flush` — The memtable-to-SSTable flush path (line 303) is where the temp-rename pattern should be applied; trace what happens to the WAL after flush
- [function] `sstable-and-compaction/sstable.py:finish` — The seek-back-to-rewrite-header technique is particularly dangerous without temp-rename since the header and footer can be inconsistent after a crash
- [general] `compaction-crash-safety` — Compaction merges multiple SSTables into new ones then deletes the originals; without atomic file creation, a crash mid-compaction can lose data from the deleted inputs before the output is durable
- [file] `hash-index-storage/bitcask.py` — The segment rotation rename at line 297 is close to the right pattern but serves a different purpose; understanding why it's there clarifies what's missing elsewhere

## Beliefs

- `sstable-writes-directly-to-final-path` — `SSTableWriter` opens and writes to the final filepath immediately; no temporary file is used, so a crash mid-write leaves a corrupt file at the production path
- `lsm-flush-lacks-fsync` — The LSM tree's `_flush` path creates SSTables without calling `os.fsync()` on either the data file or the parent directory, meaning flushed data can be lost even after the WAL is truncated
- `wal-flush-is-python-only` — The LSM WAL's `self._fd.flush()` at `lsm.py:26` pushes to OS buffers but does not call `os.fsync()`, so WAL entries are not guaranteed durable on disk
- `bitcask-rename-is-rotation-not-safety` — The `os.rename` calls in both Bitcask implementations rename active segments to frozen segments for namespace management, not for crash-safe atomic file creation
- `sstable-header-rewrite-is-crash-vulnerable` — `SSTableWriter.finish()` seeks back to offset 0 to rewrite the entry count after writing the body and index, creating a window where header and footer are inconsistent

