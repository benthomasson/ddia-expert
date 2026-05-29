# Topic: How `O_DIRECT` bypasses the page cache entirely and gives the application direct control over write alignment and sector-boundary atomicity

**Date:** 2026-05-29
**Time:** 07:57

# `O_DIRECT` and Page Cache Bypass

## The Short Answer: This Codebase Doesn't Use It

The observations are clear: **there are zero uses of `O_DIRECT` anywhere in this codebase**. Both the `O_DIRECT` pattern search and the `os.open`/`os.O_` flag search returned 0 matches. Every storage engine here uses Python's standard buffered `open()` and routes all I/O through the kernel page cache.

## What `O_DIRECT` Does (Conceptually)

`O_DIRECT` is a flag passed to the POSIX `open()` syscall that tells the kernel: *don't buffer this file's reads or writes in the page cache — transfer data directly between my application buffer and the storage device*. This has three consequences:

1. **No double-buffering** — the application manages its own cache (like a database buffer pool), and the kernel doesn't maintain a redundant copy.
2. **Alignment constraints** — the application must align its I/O buffers and offsets to the device's sector size (typically 512B or 4096B). The kernel page cache normally hides this from you.
3. **Sector-boundary atomicity** — because writes go directly to the device in sector-sized units, a single sector write is atomic from the device's perspective. With the page cache in between, the kernel batches and reorders writeback in ways the application can't control.

## What This Codebase Does Instead

Every storage engine here takes the page-cache-mediated path and enforces durability with `flush()` + `os.fsync()`:

**B-Tree (`b-tree-storage-engine/btree.py`)** — `PageManager` opens files with `open(file_path, 'r+b')` (line 33) and calls `self._f.flush()` after every page write (lines 48, 80). The `sync()` method (line 107) calls `os.fsync(self._f.fileno())`. The WAL class (line 131) does the same: `os.fsync(self._f.fileno())` after every `log_write` (line 143).

**Bitcask (`hash-index-storage/bitcask.py`)** — `_write_record` (line 87) appends to the active file, calls `self.active_file.flush()` (line 95), then conditionally calls `os.fsync(self.active_file.fileno())` (line 97) based on the `sync_writes` flag.

**WAL (`write-ahead-log/wal.py`)** — `_do_sync` (line 120) calls `self._fd.flush()` then `os.fsync(self._fd.fileno())` in `"sync"` mode. In `"batch"` mode, it delays fsync until a configurable write count threshold is hit (line 126).

**LSM Tree (`log-structured-merge-tree/lsm.py`)** — Uses `self._fd.flush()` (line 26) and has a `_flush()` method (line 303) for memtable-to-SSTable persistence.

## Why This Matters

The `flush()` + `fsync()` pattern these engines use says: *push my writes through the page cache to stable storage*. The data still passes through the kernel's page cache — it's just that `fsync` forces the kernel to write it all the way to disk before returning. This is sufficient for crash safety but not for eliminating double-buffering or controlling write ordering at the sector level.

A production database like PostgreSQL or MySQL/InnoDB uses `O_DIRECT` precisely because it already maintains its own buffer pool and doesn't want the kernel duplicating that work. These reference implementations skip that complexity entirely — they trust the page cache and use `fsync` as their durability primitive.

## What's Missing From This Codebase

To demonstrate `O_DIRECT` properly, you'd need:

- **`os.open()` with `os.O_DIRECT`** instead of Python's `open()` builtin
- **Aligned memory buffers** (e.g., via `mmap` or `ctypes`-allocated memory aligned to 4096 bytes)
- **Sector-aligned offsets** — all seeks and write sizes must be multiples of the sector size
- **An application-level buffer pool** — since the page cache is bypassed, the application must cache hot pages itself

None of these patterns exist here. This is a pedagogically appropriate trade-off: the implementations focus on the logical structure of storage engines (WAL, B-Tree splits, LSM compaction, Bitcask hash index) rather than the low-level I/O optimizations that production systems layer on top.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:PageManager` — The closest analog to a database buffer pool; understanding its fixed-size page I/O is the prerequisite for understanding why `O_DIRECT` would replace it
- [function] `write-ahead-log/wal.py:_do_sync` — Implements three sync modes (sync/batch/none) that trade durability for throughput — the same trade-off space `O_DIRECT` operates in
- [general] `o-direct-aligned-io` — How `O_DIRECT` requires `posix_memalign` or `mmap` for buffer allocation, and why Python's `open()` can't satisfy these constraints without ctypes
- [function] `hash-index-storage/bitcask.py:_write_record` — Append-only writes with optional fsync; a natural candidate for `O_DIRECT` since each record could be padded to sector boundaries
- [general] `fsync-vs-fdatasync-vs-o-direct` — The three-way trade-off between metadata durability (`fsync`), data-only durability (`fdatasync`), and cache bypass (`O_DIRECT`) that DDIA Chapter 3 discusses

## Beliefs

- `no-o-direct-usage` — No file in the ddia-implementations codebase uses `O_DIRECT`, `os.open()`, or any `os.O_` flags; all I/O goes through Python's buffered `open()` and the kernel page cache
- `fsync-is-durability-primitive` — Every storage engine (B-Tree, WAL, Bitcask, LSM) relies on `flush()` + `os.fsync()` as the sole mechanism for forcing data to stable storage
- `wal-three-sync-modes` — `WriteAheadLog._do_sync` supports three sync strategies: per-write fsync (`"sync"`), batched fsync every N writes (`"batch"`), and no fsync (`"none"`)
- `btree-page-writes-unbuffered-by-app` — `PageManager` has no application-level page cache — every `read_page` reads from the file and every `write_page` writes immediately, relying entirely on the kernel page cache for performance

