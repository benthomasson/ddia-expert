# Topic: How the OS can reorder writes within a single `write()` call and why `fdatasync` vs `fsync` matters for metadata vs data durability

**Date:** 2026-05-29
**Time:** 10:02

# Write Reordering and `fsync` vs `fdatasync` in the DDIA Implementations

## The Core Problem: What Happens Between `write()` and Disk

When you call `write()`, the data doesn't go directly to the disk platter or SSD. It passes through multiple layers that can reorder, batch, or delay the actual physical write:

1. **Userspace buffer** — Python's file object buffers in the process
2. **Kernel page cache** — the OS holds "dirty pages" in RAM
3. **Disk controller write cache** — the drive itself may reorder for seek optimization

Within a single `write()` call that spans multiple filesystem blocks (e.g., writing 8KB to a 4KB-block filesystem), the kernel is free to flush those blocks to disk in any order. A crash between block writes leaves a partially-written record on disk — some bytes from the new write, some from whatever was there before.

This is why every durability-critical path in this codebase follows the same two-step ritual: `flush()` then `fsync()`.

## How This Codebase Handles It

### The Correct Pattern: `flush()` + `fsync()`

The WAL (`write-ahead-log/wal.py:128-133`) demonstrates the canonical approach:

```python
def _do_sync(self, force: bool = False):
    if self._sync_mode == "sync" or force:
        self._fd.flush()          # push from Python buffer → kernel page cache
        os.fsync(self._fd.fileno())  # push from kernel page cache → stable storage
```

`flush()` alone only moves data from the Python runtime buffer into the kernel's page cache. The data is still in RAM — a power loss kills it. `os.fsync()` forces the kernel to write all dirty pages for that file descriptor to the physical device **and wait for the device to confirm**.

This pattern appears consistently across the durable engines:

- **Bitcask** (`hash-index-storage/bitcask.py:87-88`): `flush()` then `fsync()` on every write when `sync_writes=True`
- **B-tree WAL** (`b-tree-storage-engine/btree.py:137`): `fsync()` after every WAL entry before acknowledging
- **B-tree PageManager** (`b-tree-storage-engine/btree.py:105`): `sync()` method does `flush()` + `fsync()`

### The Durability Gap: `flush()` Without `fsync()`

Two implementations **only call `flush()`** with no `fsync()`:

- **LSM WAL** (`log-structured-merge-tree/lsm.py:26`): `self._fd.flush()` — no `fsync()` follows
- **Log-structured hash table** (`log-structured-hash-table/bitcask.py:148`): `self._active_file.flush()` — no `fsync()` follows

This means their writes survive a Python crash (data is in the kernel) but **not a power failure or OS crash** (data may still be in the page cache). For the LSM tree this is especially concerning because its WAL exists specifically to provide crash recovery — without `fsync()`, the WAL can't guarantee that either.

## Why `fsync` vs `fdatasync` Matters

`os.fsync()` flushes **both the file data and the file metadata** (size, modification time, allocation maps) to disk. `os.fdatasync()` flushes **only the data**, skipping metadata updates that aren't needed to read the file correctly.

The distinction matters because metadata updates require additional disk I/O — often to a different location on disk (the inode table). For an append-only log that writes thousands of records per second, each `fsync()` forces an extra metadata write (the file size changed) that `fdatasync()` could skip if the file was pre-allocated.

**This entire codebase uses `os.fsync()` exclusively — `fdatasync` appears zero times across all 13 sync call sites.** This is the safe default: `fsync()` is never wrong, just sometimes slower. But there's a case where it matters:

- **Appending to the WAL** (`write-ahead-log/wal.py:128`): Every append changes the file size, so `fsync()` must flush the updated size metadata too. If the metadata isn't durable and we crash, the OS might recover the old file size and silently truncate the data we thought was safe. Here, `fsync()` is correct.
- **Overwriting existing pages in the B-tree data file** (`b-tree-storage-engine/btree.py:105`): When overwriting a page in place, the file size doesn't change. `fdatasync()` would suffice and skip the redundant metadata flush — a potential performance win on every page write.

The B-tree's `PageManager.sync()` at line 105 and `close()` at lines 112-113 both use `fsync()` for in-place page overwrites where `fdatasync()` would be sufficient, since the file size hasn't changed.

## The Ordering Contract

The critical invariant across all engines: **the WAL record must be durable before the main data structure is modified**. The B-tree makes this explicit — `WAL.log_write()` (`btree.py:137`) calls `fsync()` on the WAL file, and only after that does the caller write to the data file. If the machine crashes between those two operations, recovery replays the WAL entries (`WAL.recover()` at line 144+) and reapplies the page writes.

Without `fsync()` on the WAL, you could lose both the WAL entry and the incomplete data write, leaving the database silently corrupted with no recovery path.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:WAL.commit` — Shows the WAL commit protocol: sync the data file first, then truncate the WAL — ordering these two fsyncs incorrectly would lose committed data
- [function] `write-ahead-log/wal.py:append_batch` — Demonstrates atomic batch writes: multiple records plus a COMMIT marker written in a single `write()` call, then fsynced — a crash mid-write produces a partial batch that recovery can detect and discard
- [file] `log-structured-merge-tree/lsm.py` — The LSM WAL only calls `flush()` without `fsync()`, making it the weakest durability guarantee in the codebase — compare with the standalone WAL module
- [general] `directory-fsync-after-file-creation` — None of these implementations `fsync()` the parent directory after creating new segment/WAL files, which means a crash could lose the directory entry even if the file's data is durable
- [function] `hash-index-storage/bitcask.py:_write_record` — Compare with `log-structured-hash-table/bitcask.py:_write_record` — same Bitcask design, but one fsyncs and the other doesn't, illustrating how the same algorithm can have completely different durability properties

## Beliefs

- `lsm-wal-missing-fsync` — The LSM tree WAL (`log-structured-merge-tree/lsm.py:26`) calls `flush()` but never `fsync()`, so it does not guarantee durability across power failures
- `codebase-uses-fsync-not-fdatasync` — All 13 sync call sites across the codebase use `os.fsync()`; `os.fdatasync()` is never used, even for in-place page overwrites where it would suffice
- `btree-wal-fsync-before-data-write` — The B-tree WAL (`btree.py:137`) calls `fsync()` on each WAL entry before the corresponding page is written to the data file, enforcing write-ahead ordering
- `no-directory-fsync-after-file-creation` — No implementation in the codebase fsyncs the parent directory after creating new WAL or segment files, leaving a window where the file exists on disk but isn't reachable through the directory after crash recovery

