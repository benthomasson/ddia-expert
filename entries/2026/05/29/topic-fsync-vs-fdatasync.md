# Topic: This implementation uses `os.fsync` which also syncs file metadata; `fdatasync` would be faster on Linux by skipping metadata — worth understanding the tradeoff

**Date:** 2026-05-29
**Time:** 12:11

## `os.fsync` vs `fdatasync`: The Metadata Sync Tradeoff

This codebase uses `os.fsync` exclusively — 13 call sites across three modules, zero uses of `fdatasync`. That's a deliberate choice favoring correctness over performance, but it's worth understanding what you're paying for.

### What `fsync` actually does

`os.fsync(fd)` forces both the **file data** and **file metadata** (size, modification time, permissions, directory entry) from the OS page cache to durable storage. The kernel is allowed to buffer writes indefinitely; without an explicit sync, a power failure can lose data that `write()` already "succeeded" on.

### Where this codebase calls it

The calls fall into three categories:

**WAL durability** (`write-ahead-log/wal.py`): The `_do_sync` method at lines 127–133 calls `os.fsync` after every write in `"sync"` mode, or every N writes in `"batch"` mode. The `_rotate` method (line 115) syncs before closing the old file. `truncate` (line 184) syncs before rewriting. These are the most performance-critical call sites — WAL append is on the hot path for every write operation.

**B-tree page manager** (`b-tree-storage-engine/btree.py`): `PageManager.sync()` at line 105 and `PageManager.close()` at line 113 both flush+fsync. The WAL's `log_write` (line 137), `commit` (line 144), and `recover` (line 171) also fsync. Every page write goes through the WAL first, so you're paying for fsync twice per mutation: once for the WAL entry, once when committing pages to the data file.

**Bitcask** (`hash-index-storage/bitcask.py`): `_write_record` at line 88 syncs after every record when `sync_writes=True` (the default). Every `put` and `delete` hits this path.

### What `fdatasync` would save

`fdatasync` (available as `os.fdatasync` on Linux, not available on macOS) skips syncing file metadata **unless the metadata is needed to locate the data** — specifically, it still syncs the file size if it changed (because you need the size to find the end of the file), but skips things like `mtime` and `atime`.

The practical saving: one fewer disk I/O operation per sync in cases where the file size didn't change (overwriting existing pages in the B-tree) or where the metadata update can be deferred. On spinning disks, this can save a full disk revolution (~8ms). On SSDs the difference is smaller but still measurable under high throughput.

### Why `fsync` is the right default here

For a **reference implementation** of DDIA concepts, `fsync` is the correct choice:

1. **Portability**: `os.fdatasync` doesn't exist on macOS or Windows. These implementations run on all platforms without conditional logic.

2. **Append-heavy workloads**: The WAL (`wal.py`) and Bitcask (`bitcask.py`) are append-only. Every write extends the file, changing its size. `fdatasync` **must** sync the size change in this case, so the saving is minimal — you'd only skip the `mtime` update.

3. **B-tree overwrites are the exception**: `PageManager.write_page` (btree.py:80) overwrites existing pages in-place. This is the one case where `fdatasync` would meaningfully help — the file size doesn't change, so metadata sync is pure overhead. But the B-tree WAL writes (which are appends) still wouldn't benefit.

4. **Correctness over performance**: The LSM tree's WAL (`log-structured-merge-tree/lsm.py`) is notable for calling `flush()` at line 26 **without** any `fsync` at all — it's the least durable module. The other modules err on the side of too much syncing rather than too little, which is appropriate for teaching crash recovery semantics.

### The real optimization opportunity

The bigger performance lever isn't `fsync` vs `fdatasync` — it's **sync frequency**. The WAL already demonstrates this with its `sync_mode` parameter: `"sync"` (every write), `"batch"` (every N writes), or `"none"` (never). Batching amortizes the cost of one `fsync` across many writes. The Bitcask store's `sync_writes` boolean at construction time is the same idea. Group commit — collecting multiple writers' data and issuing one fsync for the batch — is what production databases actually do to solve this.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_do_sync` — The sync policy switch (`sync`/`batch`/`none`) is the most interesting durability-vs-performance knob in the codebase; trace how `batch_sync_count` changes the crash window
- [function] `log-structured-merge-tree/lsm.py:WAL.append` — This WAL calls `flush()` without `fsync`, making it the least durable implementation; compare its crash guarantees to the other WALs
- [general] `group-commit-pattern` — How production databases (PostgreSQL, SQLite) amortize fsync cost across concurrent transactions, which is a bigger win than fdatasync
- [function] `b-tree-storage-engine/btree.py:WAL.log_write` — The B-tree pays for fsync twice per mutation (WAL + data file commit); understand why the double-sync is necessary for crash safety
- [file] `hash-index-storage/bitcask.py` — The `sync_writes` constructor flag lets callers opt out entirely; explore what guarantees remain when sync is disabled

## Beliefs

- `fsync-exclusive-no-fdatasync` — All 13 sync points across the codebase use `os.fsync`; `fdatasync` is never used, prioritizing portability and correctness over Linux-specific optimization
- `wal-sync-mode-controls-durability-window` — The WAL's `sync_mode` parameter (`sync`/`batch`/`none`) determines how many records can be lost on crash: zero, up to `batch_sync_count`, or unbounded
- `lsm-wal-flush-without-fsync` — The LSM tree's WAL (`lsm.py:26`) calls `flush()` but never `fsync`, meaning its writes survive process crashes but not power failures
- `btree-double-fsync-per-mutation` — B-tree mutations pay for `fsync` twice: once when writing the WAL entry (`btree.py:137`) and again when committing the page to the data file (`btree.py:105`), with the WAL cleared only after the data file is confirmed durable
- `bitcask-per-record-fsync-default` — Bitcask syncs after every individual record write by default (`sync_writes=True`), making it the most durable but slowest module for single-key writes

