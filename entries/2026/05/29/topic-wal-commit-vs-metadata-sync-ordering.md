# Topic: The WAL `commit` method calls `page_manager.sync()` then truncates the WAL; trace whether metadata is ever synced as part of this commit fence

**Date:** 2026-05-29
**Time:** 10:31

## WAL `commit` and the Metadata Sync Question

The WAL's `commit` method lives at `b-tree-storage-engine/btree.py:139`:

```python
def commit(self, page_manager):
    page_manager.sync()
    self._f.seek(0)
    self._f.truncate(0)
    self._f.flush()
    os.fsync(self._f.fileno())
    self._seq = 0
```

The commit fence has two phases:

1. **Sync the data file** — `page_manager.sync()` (line 103) calls `flush()` then `os.fsync()` on the single data file (`self._f`). This forces every dirty page — including any metadata page writes — to stable storage.

2. **Truncate the WAL** — only after the fsync returns does the WAL zero itself out and fsync its own file descriptor.

### Does metadata get synced?

**Yes, but only implicitly.** The `PageManager` uses a single file handle (`self._f`) for everything — data pages *and* the metadata page (page 0). When `sync()` calls `os.fsync(self._f.fileno())`, the kernel flushes all pending writes to that file descriptor, which includes any metadata writes that happened earlier.

Here's the key: look at how metadata is written. `PageManager._write_meta()` (line 42) does a `seek(0)` + `write()` + `flush()`, but **no fsync**. The same is true for `write_meta()` (line 60), which delegates to `_write_meta()`. So during normal B-tree operations — splits, allocations via `allocate_page()` (line 87), frees via `free_page()` (line 97) — metadata is written to the OS page cache but not forced to disk.

The metadata only hits disk when one of these happens:
- `page_manager.sync()` — which is exactly what `WAL.commit()` calls
- `page_manager.close()` (line 108) — which also fsyncs

### The durability contract

The ordering is:

```
WAL.log_write()  →  fsync WAL entry
 ... (more writes) ...
WAL.commit()     →  fsync data file (including metadata page 0)
                 →  truncate + fsync WAL file
```

This is correct for crash recovery: if the process crashes between `log_write` and `commit`, the WAL entries survive and `recover()` replays them. If it crashes after the data-file fsync but before the WAL truncate, recovery replays already-applied writes — which is idempotent since `write_page` overwrites the same page numbers.

### The gap

There is **no separate metadata sync** — metadata is flushed alongside all other pages in the single `os.fsync()` call. This works because `PageManager` uses one file for everything. If the design ever split metadata into a separate file, the commit fence would need an additional fsync for that file, and the ordering between the two fsyncs would matter.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:allocate_page` — Mutates metadata (next_free, free_head) without fsync; understand which operations leave metadata dirty before a commit fence
- [function] `b-tree-storage-engine/btree.py:recover` — Traces the recovery path that replays WAL entries; check whether metadata consistency is restored or assumed
- [general] `fsync-ordering-guarantees` — Whether the OS guarantees write ordering within a single fsync call, and what happens if metadata page 0 and a data page are both dirty
- [function] `b-tree-storage-engine/btree.py:log_write` — Each WAL entry is individually fsynced; trace whether this creates a performance bottleneck during splits that touch multiple pages
- [file] `write-ahead-log/wal.py` — The standalone WAL module has a different design (record-level, not page-level); compare its truncation strategy with the B-tree WAL's full-file truncate

## Beliefs

- `wal-commit-syncs-metadata-implicitly` — `WAL.commit()` syncs metadata page 0 because `PageManager.sync()` fsyncs the single shared file descriptor; there is no dedicated metadata fsync
- `write-meta-no-fsync` — `PageManager._write_meta()` calls `flush()` but never `fsync()`; metadata durability depends entirely on a later `sync()` or `close()` call
- `commit-fence-ordering` — The WAL commit fence guarantees data-file fsync completes before the WAL is truncated, making recovery replay idempotent
- `single-file-design-enables-atomic-sync` — `PageManager` stores metadata and data pages in one file, so a single `os.fsync()` covers both; splitting into separate files would break the commit fence

