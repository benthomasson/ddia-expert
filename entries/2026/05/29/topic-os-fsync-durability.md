# Topic: Neither `truncate` nor `_flush` calls `fsync()`, meaning data could be lost in an OS crash even after the Python-level write completes

**Date:** 2026-05-29
**Time:** 11:19

# The `flush()` vs `fsync()` Durability Gap

## The Core Problem

When Python calls `file.flush()`, it pushes data from Python's internal buffer into the **operating system's page cache** — but the data hasn't necessarily reached the physical disk yet. Only `os.fsync(fd)` forces the OS to write dirty pages to durable storage. If the machine loses power between `flush()` and the OS eventually writing the page to disk, that data is gone.

## Where the Gap Exists

### LSM Tree WAL — `log-structured-merge-tree/lsm.py`

This is the clearest example. The WAL's `append()` (line 21) writes records and calls only `self._fd.flush()` at line 27 — **no `os.fsync()`**:

```python
def append(self, key: str, value: bytes):
    k = key.encode("utf-8")
    self._fd.write(struct.pack(">I", len(k)))
    self._fd.write(k)
    self._fd.write(struct.pack(">I", len(value)))
    self._fd.write(value)
    self._fd.flush()          # ← Python → OS buffer only
```

The `truncate()` method (line 56) is even worse — it reopens the file to clear it, with no sync at all:

```python
def truncate(self):
    self._fd.close()
    self._fd = open(self._path, "wb")   # truncates to zero
    self._fd.close()                     # no fsync before close
    self._fd = open(self._path, "ab")
```

This creates a concrete crash scenario: the LSM tree flushes its memtable to an SSTable, then calls `self._wal.truncate()` (line 314) to clear the WAL. If the OS has written the truncation to disk but *not* the SSTable data, the entries in that memtable are permanently lost — the WAL is empty and the SSTable is incomplete.

### B-Tree PageManager — `b-tree-storage-engine/btree.py`

The `PageManager` has a similar pattern. `_write_meta()` (line 48), `_write_empty_leaf()` (line 55), and `write_page()` (line 81) all end with `self._f.flush()` but no `fsync()`. The class *does* have an explicit `sync()` method (line 104) and `close()` (line 111) that call `os.fsync()`, but individual page writes during normal operation don't.

This is actually **intentional** in the B-tree — the WAL (`btree.py:137`, line 137) calls `os.fsync()` after every logged write, and `commit()` calls `page_manager.sync()` before clearing the WAL. The PageManager defers durability to batch syncs, while the WAL guarantees recoverability. The gap is deliberate because the WAL is the durability mechanism, not the data file.

### Event Store — `event-sourcing-store/event_store.py`

`_persist_event()` (line 131) is the most extreme case — it opens the file in a `with` block, writes JSON, and relies on the implicit `close()` to push data out. No `flush()`, no `fsync()`. Python's `close()` flushes the Python buffer, but the OS can still lose the data before it hits disk.

## Where It's Done Correctly

The standalone **write-ahead-log module** (`write-ahead-log/wal.py`) is the gold standard in this codebase. Its `_do_sync()` method (line 122) calls both `self._fd.flush()` and `os.fsync(self._fd.fileno())`. Every critical operation — `append` (via `_do_sync`), `append_batch`, `checkpoint`, `truncate` (line 184), and `rotate` — calls `fsync()`. Even **Bitcask** (`hash-index-storage/bitcask.py:88`) conditionally calls `os.fsync()` in `_write_record()` when `sync_writes` is enabled.

## Why It Matters

A write-ahead log that doesn't actually survive crashes isn't a write-ahead log — it's a write-behind-hope-for-the-best log. The entire point of WAL is that the log is durable *before* the main data structure is updated, so you can replay it after a crash. The LSM tree's WAL violates this contract: a power failure at the wrong moment loses acknowledged writes silently.

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:_flush` — The memtable-to-SSTable flush path; check whether SSTable writes also lack fsync, compounding the durability gap
- [function] `write-ahead-log/wal.py:_do_sync` — The correct fsync pattern with configurable sync modes (sync vs batch) that the LSM WAL should emulate
- [function] `b-tree-storage-engine/btree.py:commit` — How the B-tree WAL coordinates fsync ordering between the WAL and the data file to ensure crash safety
- [general] `fdatasync-vs-fsync` — `fdatasync()` skips metadata updates and is faster on Linux; relevant for high-write workloads where the LSM WAL sync gap is most painful
- [file] `event-sourcing-store/event_store.py` — The event store's persistence has no durability guarantees at all; contrast with the WAL module's approach

## Beliefs

- `lsm-wal-no-fsync` — The LSM tree's WAL (`log-structured-merge-tree/lsm.py`) calls `flush()` but never `os.fsync()`, so acknowledged writes can be lost on OS crash
- `lsm-truncate-before-sstable-sync` — The LSM tree truncates the WAL after flushing memtable to SSTable, but neither operation calls fsync, creating a window where both the WAL and SSTable data exist only in OS page cache
- `wal-module-syncs-every-write` — The standalone WAL module (`write-ahead-log/wal.py`) calls `os.fsync()` after every append in sync mode and every N writes in batch mode, making it the only WAL implementation with real crash durability
- `btree-wal-fsync-pagefile-deferred` — The B-tree's WAL calls `fsync()` on every log entry, but the PageManager defers fsync to explicit `sync()` or `close()` calls — a deliberate design where the WAL provides crash recovery for un-synced page writes
- `event-store-persist-no-durability` — `EventStore._persist_event()` writes to disk with no `flush()` or `fsync()`, relying entirely on OS-level buffering and implicit close behavior

