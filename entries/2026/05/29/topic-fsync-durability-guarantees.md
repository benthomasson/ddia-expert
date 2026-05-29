# Topic: Explore what `os.fsync` actually guarantees on different filesystems (ext4, XFS, APFS) and why `flush()` alone is insufficient

**Date:** 2026-05-29
**Time:** 07:29

# What `os.fsync` Guarantees and Why `flush()` Alone Is Insufficient

## The Two-Layer Buffer Problem

When you write to a file in Python, data passes through **two distinct buffers** before reaching persistent storage:

1. **Language-level buffer** — Python's `io` library holds bytes in a userspace buffer (the `BufferedWriter` wrapping the file descriptor). `flush()` pushes this buffer into the kernel.
2. **OS/kernel buffer** — The kernel's page cache holds the data in RAM. `os.fsync()` forces the kernel to push this buffer to the physical storage device.

Calling `flush()` alone only clears layer 1. The data is now in the kernel's page cache — still volatile RAM. If the process crashes after `flush()` but the OS stays up, the kernel will eventually write it out. But if the **machine loses power** or the **kernel panics**, anything sitting in the page cache is lost.

## How This Codebase Handles It

The pattern is consistent across all durability-critical paths: **`flush()` immediately followed by `os.fsync()`**.

### Write-Ahead Log (`write-ahead-log/wal.py`)

The `_do_sync` method at line 124 shows the deliberate pairing:

```python
def _do_sync(self, force: bool = False):
    if self._sync_mode == "sync" or force:
        self._fd.flush()          # line 127: userspace → kernel
        os.fsync(self._fd.fileno())  # line 128: kernel → disk
    elif self._sync_mode == "batch":
        self._write_count += 1
        if self._write_count >= self._batch_sync_count:
            self._fd.flush()          # line 132
            os.fsync(self._fd.fileno())  # line 133
```

The WAL offers three sync modes — `"sync"` (fsync every write), `"batch"` (fsync every N writes), and implicitly `"none"`. This is a classic durability-vs-throughput tradeoff. Batch mode accumulates up to `batch_sync_count` (default 100) writes before fsyncing, which means up to 99 records could be lost on power failure. The `force=True` parameter overrides this for critical operations like commits (`append_batch` at line 165) and checkpoints (line 180).

### B-Tree WAL (`b-tree-storage-engine/btree.py`)

The B-Tree's WAL at line 137 does the same:

```python
def log_write(self, page_num, page_data):
    ...
    self._f.write(header + page_data + checksum)
    self._f.flush()               # line 136
    os.fsync(self._f.fileno())    # line 137
```

And the `commit` method (line 140) fsyncs the data file via `page_manager.sync()`, then truncates and fsyncs the WAL itself — ensuring the data file is durable **before** the WAL is cleared. Reversing this order would create a window where both the WAL and the data file lack the update.

### Bitcask (`hash-index-storage/bitcask.py`)

The `_write_record` method at line 85-88:

```python
self.active_file.write(record)
self.active_file.flush()
if self.sync_writes:
    os.fsync(self.active_file.fileno())
```

Here `sync_writes` is a constructor parameter (default `True`), giving callers the option to trade durability for speed.

### LSM Tree — The Notable Exception (`log-structured-merge-tree/lsm.py`)

The LSM tree's WAL at line 26 calls **only `flush()`** — no `fsync()`:

```python
def append(self, key: str, value: bytes):
    ...
    self._fd.flush()
```

This means the LSM WAL relies on the kernel to eventually write data out. On a clean process crash, the data is likely recoverable (it's in the page cache). On a power failure, recently appended WAL entries may be silently lost. This is a durability gap compared to the other implementations.

## What `fsync` Guarantees Per Filesystem

`os.fsync()` maps to the POSIX `fsync(2)` syscall, but the actual guarantee depends on the filesystem and hardware:

| Filesystem | `fsync` behavior | Gotchas |
|------------|-----------------|---------|
| **ext4** | Flushes file data and metadata to the storage device. With `data=ordered` (default mount), unwritten data blocks are flushed before the metadata journal commits. With `data=journal`, both go through the journal. Pre-4.13 kernels had a bug where `fsync` in `data=ordered` mode could reorder writes, leading to zero-length files on crash. | `fdatasync` skips metadata if only data changed — faster for append-only patterns like these WALs. |
| **XFS** | Flushes data + metadata. XFS uses delayed allocation aggressively, so without `fsync`, file data may not exist on disk at all (just metadata intent). XFS is particularly aggressive about keeping data in page cache. | Creating a new file and fsyncing it isn't enough — you must also `fsync` the **parent directory** to ensure the directory entry is durable. None of these implementations do this (e.g., `_rotate` in `wal.py` line 113-116 creates a new file without directory fsync). |
| **APFS** (macOS) | `fsync` on macOS does **not** guarantee a write-through to physical media — it may only flush to the drive's volatile write cache. For a true hardware-level guarantee, you need `fcntl(fd, F_FULLFSYNC)`. This is a critical difference for macOS development. | None of these implementations use `F_FULLFSYNC`. On macOS, even the `"sync"` mode WAL could lose data on sudden power loss. |

The codebase uses `os.fsync` everywhere (13 call sites), never `os.fdatasync` or `F_FULLFSYNC`. Since these are reference implementations for DDIA concepts rather than production systems, this is a reasonable simplification — but it means the durability guarantees are weaker than they appear on macOS, and slightly suboptimal on Linux (where `fdatasync` would suffice for append-only files).

## The Directory Fsync Problem

There's a subtlety none of these implementations address: when you create a new file (as in WAL rotation at `wal.py:116` or SSTable creation), the **file itself** can be fsynced, but the **directory entry** pointing to it may still be in the page cache. If power is lost before the directory is fsynced, the file may vanish entirely — even if its contents were fsynced. Production databases like PostgreSQL and SQLite fsync the parent directory after creating new files.

## Summary

| Operation | What it guarantees | Crash scenario survived |
|-----------|-------------------|------------------------|
| `write()` alone | Data in Python buffer | Nothing |
| `flush()` alone | Data in kernel page cache | Process crash (not power loss) |
| `flush()` + `fsync()` | Data on storage device* | Power loss* |
| `flush()` + `F_FULLFSYNC` | Data on physical media (macOS) | Power loss including drive cache |

\* Subject to filesystem and hardware write-cache behavior.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_do_sync` — Compare the three sync modes (sync/batch/none) and trace which code paths use `force=True` to understand where the implementation draws the durability line
- [function] `b-tree-storage-engine/btree.py:commit` — Study the ordering of fsync-then-truncate in WAL commit to understand why the sequence matters for crash safety
- [file] `log-structured-merge-tree/lsm.py` — Investigate the missing `fsync` in the LSM WAL and whether SSTable writes (which use plain `open`/`write`/`close`) have an implicit fsync gap
- [general] `directory-fsync-gap` — None of the implementations fsync the parent directory after creating new files (WAL rotation, SSTable creation); explore how this could cause data loss on ext4/XFS
- [general] `fdatasync-vs-fsync-optimization` — For append-only files like WALs, `fdatasync` skips unnecessary metadata updates (mtime) and can be 2-3x faster; explore whether switching would be safe here

## Beliefs

- `wal-flush-fsync-pairing` — The WAL (`wal.py`) and B-Tree WAL (`btree.py`) always call `flush()` immediately before `os.fsync()` at every sync point; one is never called without the other in sync/force modes
- `lsm-wal-no-fsync` — The LSM tree's WAL class (`lsm.py:26`) calls only `flush()` with no `os.fsync()`, making it vulnerable to data loss on power failure unlike the other WAL implementations
- `no-directory-fsync` — No implementation in the codebase fsyncs the parent directory after creating new files (WAL rotation, SSTable writes), leaving a crash window where newly created files could vanish
- `no-platform-specific-sync` — The codebase uses `os.fsync` exclusively (13 call sites) with no use of `fdatasync` or `F_FULLFSYNC`, meaning durability is weaker than advertised on macOS/APFS
- `batch-sync-commit-override` — The WAL's batch sync mode is overridden by `force=True` for commit records and checkpoints (`wal.py:165,180`), ensuring transaction boundaries are always durable regardless of sync mode

