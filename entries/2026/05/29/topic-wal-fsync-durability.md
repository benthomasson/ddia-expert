# Topic: This WAL calls `flush()` but not `os.fsync()`. Explore the difference: `flush` guarantees process-crash durability but not power-failure durability. Real databases (Postgres, SQLite) always fsync the WAL.

**Date:** 2026-05-29
**Time:** 11:41

# flush() vs. fsync(): The Durability Gap

## The Short Version

The question's premise needs a correction: **this WAL actually does call `os.fsync()`** — but only in certain sync modes, and other modules in the codebase don't. That split is where the real lesson lives.

## What the WAL Gets Right

The `_do_sync` method in `write-ahead-log/wal.py:126-134` properly pairs `flush()` with `os.fsync()`:

```python
def _do_sync(self, force: bool = False):
    if self._sync_mode == "sync" or force:
        self._fd.flush()
        os.fsync(self._fd.fileno())
    elif self._sync_mode == "batch":
        self._write_count += 1
        if self._write_count >= self._batch_sync_count:
            self._fd.flush()
            os.fsync(self._fd.fileno())
            self._write_count = 0
```

Every critical path — `append` (line 153), `append_batch` (line 167), `checkpoint` (line 174), `truncate` (line 183), and `_rotate` (line 114) — goes through this method or calls both explicitly. The WAL author understood the distinction.

## What flush() Actually Does

`flush()` pushes data from Python's **userspace buffer** (the `io.BufferedWriter` internal to the file object) into the **kernel's page cache**. After `flush()`:

- Another process on the same machine reading the file **will** see the data.
- If your **process** crashes, the data survives — the kernel still has it in its page cache and will eventually write it to disk.
- If the **machine loses power**, the page cache is gone. The data never made it to the physical storage device.

## What os.fsync() Adds

`os.fsync()` forces the kernel to write the page cache contents to the **physical storage device** (or at least to the device's write-back cache, which is battery-backed on enterprise drives). After `fsync()`:

- Even if the machine loses power, the data is on persistent storage.
- This is dramatically slower — fsync can take milliseconds, while flush takes microseconds.

The stack looks like this:

```
Python buffer  ──flush()──►  Kernel page cache  ──fsync()──►  Disk platters/NAND
  (process)                    (OS memory)                     (persistent)
```

## Where the Codebase Gets This Wrong

### The LSM Tree: flush-only, no fsync

`log-structured-merge-tree/lsm.py:26` calls `self._fd.flush()` with **no corresponding `os.fsync()`** anywhere in the file. The grep results show zero fsync hits in `lsm.py`. This means the LSM tree's WAL survives process crashes but not power failures — a real data loss risk for anything that claimed to have committed.

### The B-Tree: Mixed Behavior

`b-tree-storage-engine/btree.py` has an interesting split. Lines 46, 54, and 80 call `flush()` without `fsync()`, while lines 104-105, 112-113, 136-137, 143-144, and 170-171 properly pair them. This suggests the author deliberately chose which operations need full durability (structural changes to the tree) versus which can tolerate the page-cache-only window (perhaps read-path metadata updates).

### Bitcask: Configurable, Like the WAL

`hash-index-storage/bitcask.py:86-88` makes fsync conditional on `self.sync_writes`, mirroring the WAL's `sync_mode` approach. When enabled, every `_write_record` does `flush()` then `os.fsync()`.

## The sync_mode="none" Escape Hatch

The WAL constructor at `wal.py:71` accepts `sync_mode="none"`, which skips both flush and fsync in `_do_sync`. The `test_sync_modes` test at `test_wal.py:107-112` exercises all three modes but only checks that the code doesn't crash — it doesn't verify durability semantics. In production, `sync_mode="none"` means your WAL provides crash recovery in name only.

## How Production Databases Handle This

PostgreSQL calls `fsync()` (or `fdatasync()`) on every WAL write in its default `synchronous_commit = on` mode. SQLite in WAL mode fsyncs the WAL on every transaction commit by default (`PRAGMA synchronous = FULL`). Both offer the option to relax this for performance — PostgreSQL's `synchronous_commit = off`, SQLite's `PRAGMA synchronous = NORMAL` — but they make the tradeoff explicit and default to the safe path.

This WAL implementation gets the architecture right by exposing sync modes, but the LSM tree silently drops the guarantee, which is the more dangerous pattern.

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:_flush` — The LSM flush-to-SSTable path has no fsync; trace whether SSTable files are also unfsynced, compounding the durability gap
- [function] `write-ahead-log/wal.py:append_batch` — Atomic batch writes force fsync even in batch mode (`force=True`); explore why batch atomicity requires immediate durability
- [function] `b-tree-storage-engine/btree.py:PageManager` — Some page writes flush without fsync; understand which operations the author deemed safe to defer
- [general] `fdatasync-vs-fsync` — `fdatasync()` skips metadata (mtime, atime) updates and is faster; production databases often prefer it, but this codebase never uses it
- [general] `directory-fsync-after-rename` — Creating or renaming a file isn't durable until you fsync the *directory*; the WAL's `_rotate` fsyncs the old file but not the directory entry for the new one

## Beliefs

- `wal-fsync-in-sync-and-batch-modes` — The WAL's `_do_sync` method always pairs `flush()` with `os.fsync()` when `sync_mode` is "sync" or "batch"; only `sync_mode="none"` skips both
- `lsm-wal-has-no-fsync` — The LSM tree at `lsm.py` calls `flush()` but never `os.fsync()`, meaning it cannot survive power failure without data loss
- `btree-selective-fsync` — The B-tree's `PageManager` fsyncs structural mutations (lines 105, 113, 137, 144, 171) but calls flush-only on some other write paths (lines 46, 54, 80)
- `wal-sync-mode-none-is-unsafe` — `WriteAheadLog(sync_mode="none")` provides no durability guarantee beyond normal OS buffering; the test suite does not verify durability semantics for any sync mode

