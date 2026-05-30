# Topic: Understanding the OS page cache layer between `flush()` and physical disk writes, and how `fdatasync` differs from `fsync`

**Date:** 2026-05-29
**Time:** 12:33

# The OS Page Cache: What Happens Between `flush()` and Disk

## The Two-Stage Write Path

When this codebase writes data to disk, there are actually **three layers** a write passes through:

1. **Python's internal buffer** (userspace, in-process)
2. **OS page cache** (kernel memory, shared across processes)
3. **Physical storage** (the actual disk platters or SSD cells)

The critical insight is that `.flush()` and `os.fsync()` address *different boundaries* in this stack.

### Stage 1: `flush()` — Userspace to Kernel

Python's file objects maintain an internal buffer in the process's memory. When you call `self._fd.flush()`, you push bytes from that Python buffer into the **OS page cache** — a region of kernel-managed RAM that acts as a write-back cache for disk I/O.

After `flush()`, the data is visible to other processes reading the same file, but it is **not on disk**. A power failure at this point loses the data.

You can see this pattern throughout the codebase. For example, in the LSM tree's WAL (`log-structured-merge-tree/lsm.py:26`):

```python
self._fd.flush()
```

This WAL only calls `flush()` — it never calls `os.fsync()`. That means **the LSM WAL does not guarantee durability on its own**. Data could be lost in a crash. This is a deliberate trade-off: the LSM tree prioritizes write throughput over per-record durability.

### Stage 2: `os.fsync()` — Kernel to Physical Disk

`os.fsync()` tells the OS to force all dirty pages for the given file descriptor from the page cache to stable storage. It blocks until the disk controller acknowledges the write is persistent. This is expensive — it may take milliseconds, requiring a physical disk seek or an SSD flush of its volatile write buffer.

The WAL module (`write-ahead-log/wal.py:124-134`) shows the canonical two-step pattern:

```python
def _do_sync(self, force: bool = False):
    if self._sync_mode == "sync" or force:
        self._fd.flush()              # step 1: Python buffer → page cache
        os.fsync(self._fd.fileno())   # step 2: page cache → disk
    elif self._sync_mode == "batch":
        self._write_count += 1
        if self._write_count >= self._batch_sync_count:
            self._fd.flush()
            os.fsync(self._fd.fileno())
            self._write_count = 0
```

**You must call `flush()` before `fsync()`**. If you only call `fsync()`, bytes still sitting in Python's internal buffer won't reach the kernel, so `fsync()` has nothing new to push to disk.

### Where Each Module Sits on the Durability Spectrum

| Module | `flush()` | `os.fsync()` | Durability |
|--------|-----------|-------------|------------|
| LSM WAL (`lsm.py:26`) | Yes | **No** | Page cache only — crash-vulnerable |
| Write-ahead log (`wal.py:115`) | Yes | Yes (configurable) | Full durability in `sync` mode |
| B-tree PageManager (`btree.py:104-105`) | Yes | Yes (on `sync()` and `close()`) | Full durability on explicit sync |
| B-tree WAL (`btree.py:136-137`) | Yes | Yes (every log entry) | Full durability per write |
| Bitcask (`bitcask.py:86-88`) | Yes | Conditional (`sync_writes` flag) | Configurable |

## `fsync` vs `fdatasync`

This codebase uses **only `os.fsync()`** — there are zero calls to `os.fdatasync()` anywhere. Here's the difference:

- **`fsync`** flushes both the **file data** and the **file metadata** (size, modification time, permissions) to disk. This requires *two* writes to different locations on disk: one for the data blocks, one for the inode.

- **`fdatasync`** flushes only the **file data** and only the metadata that is *necessary to read the data back* (primarily the file size, if it changed). It skips metadata like `mtime` and `atime`. This can be up to **2x faster** because it avoids an extra disk seek to the inode.

The practical distinction matters most for append-only logs like this codebase's WAL. When appending records, the file size changes with every write, so `fdatasync` still updates that. But it skips the `mtime` update, saving one disk operation per sync. For a WAL in `sync` mode that calls `fsync` on every single append (`wal.py:115`), switching to `fdatasync` could nearly double write throughput.

The reason the codebase uses `fsync` everywhere is likely simplicity and portability — `fdatasync` is not available on all platforms (notably, Python's `os.fdatasync` exists on Linux but not macOS).

## The Sync Mode Trade-Off

The WAL's configurable `sync_mode` (`wal.py:64-70`) illustrates the fundamental durability vs. performance trade-off:

- **`"sync"`**: `flush()` + `fsync()` on every write. Maximum durability, lowest throughput.
- **`"batch"`**: `flush()` + `fsync()` every N writes (default 100). Loses up to N-1 records on crash, but dramatically higher throughput.
- **`"none"`** (implicit): Only `flush()`, no `fsync()`. Fastest, but relies entirely on the OS flushing the page cache before a crash — typically within 30 seconds on Linux (controlled by `vm.dirty_expire_centisecs`), but not guaranteed.

The B-tree WAL (`btree.py:136-137`) takes no chances — it syncs on every single `log_write`, because a B-tree mutation that's half-applied is structurally corrupt.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_do_sync` — The central durability decision point; trace how `sync_mode` and `force` interact across `append`, `append_batch`, and `checkpoint`
- [function] `b-tree-storage-engine/btree.py:PageManager.sync` — Compare the B-tree's sync strategy (explicit calls) with the WAL's per-record sync; notice that `write_page` only flushes but doesn't fsync
- [general] `page-cache-write-ordering` — Explore what guarantees the OS gives about write ordering within the page cache, and why the B-tree WAL writes the log entry *before* the data page
- [function] `write-ahead-log/wal.py:truncate` — Examine why truncation does flush+fsync+close before rewriting files, and what could go wrong if it didn't
- [general] `fdatasync-portability` — Investigate `os.fdatasync` availability across platforms and why databases like PostgreSQL use `fdatasync` on Linux but fall back to `fsync` on macOS

## Beliefs

- `lsm-wal-no-fsync` — The LSM tree's WAL (`lsm.py`) calls only `flush()` and never `os.fsync()`, meaning it does not guarantee durability against OS crashes or power loss
- `flush-before-fsync-invariant` — Every call to `os.fsync()` in the codebase is immediately preceded by a `.flush()` call, ensuring Python's userspace buffer is drained to the kernel before the disk sync
- `btree-wal-sync-every-write` — The B-tree WAL (`btree.py:136-137`) calls `flush()` + `os.fsync()` on every `log_write`, providing per-operation durability regardless of configuration
- `no-fdatasync-usage` — The entire codebase uses `os.fsync()` exclusively; `os.fdatasync()` is never used, trading potential write performance for simplicity and cross-platform compatibility
- `wal-sync-mode-configurable` — The standalone WAL module supports three sync modes (`sync`, `batch`, `none`) controlled at construction time, but `append_batch` and `checkpoint` always force a sync regardless of mode

