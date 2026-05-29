# Topic: The `_do_sync` method (lines 126-134) treats batches specially with `force=True` — explore what happens to individual appends under `batch` or `none` sync modes

**Date:** 2026-05-29
**Time:** 06:34

The observations provided cover the key code. Let me trace through the full sync behavior.

---

## How `_do_sync` handles individual appends under different sync modes

### The three-way branch in `_do_sync` (`write-ahead-log/wal.py:122-134`)

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

The `force` parameter is the key. When `force=True`, the method **always** flushes and fsyncs regardless of mode — it short-circuits through the first branch (`self._sync_mode == "sync" or force`). This is the mechanism batch write methods use to guarantee durability at batch boundaries.

### What happens to individual appends

For a single `append()` call (`wal.py:141`), the behavior depends entirely on which sync mode was configured at `__init__` (`wal.py:63`):

**`sync_mode="sync"` (default):** Every `append` calls `_do_sync()` without `force=True`, but the first condition `self._sync_mode == "sync"` is true, so it flushes and fsyncs. Every write is individually durable. Maximum safety, worst throughput.

**`sync_mode="batch"`:** Individual appends call `_do_sync()` without `force=True`. The first condition is false (mode isn't `"sync"` and force isn't true), so it falls into the `elif` branch. The write counter increments, and an fsync only happens when `_write_count >= self._batch_sync_count` (configured at init, default 100 per `wal.py:65`). This means **up to 99 appends sit only in the OS page cache with no fsync guarantee**. A crash during that window loses those records — they were written to the file via `_fd.write()`, but without fsync the OS can discard them.

**`sync_mode="none"`:** Neither condition matches — `"none" != "sync"`, force is `False`, and `"none" != "batch"`. The method **does nothing**. No flush, no fsync, ever, for individual appends. Data lives only in Python's write buffer and/or the OS page cache until the file is rotated (the `_rotate` method at line 113 does an unconditional `flush()` + `fsync()` before closing) or the WAL is explicitly closed. This is the fastest mode but can lose an arbitrary number of recent writes on crash.

### Why `force=True` exists

Batch write methods (likely a `write_batch` or similar — the observations don't show the full method but we can see `append` at line 141) need a way to say "I've written N records, now make them all durable at once." By passing `force=True`, they bypass the mode check entirely. This gives batch callers the guarantee: "all records written in this batch are now on disk," regardless of whether the WAL is in `batch`, `none`, or `sync` mode.

This is the classic **group commit** pattern from DDIA Chapter 3: amortize the cost of fsync across many writes rather than paying it per-record.

### The durability gap in `batch` mode

There's a subtle asymmetry worth noting. In `batch` mode, the counter `_write_count` is **not reset** when a forced sync happens (only the counter-triggered sync at line 133 resets it). So if a batch write does `force=True`, the counter still reflects stale state from prior individual appends. Whether this is a bug or intentional depends on whether the counter should track "appends since last sync" (any sync) or "appends since last batch-triggered sync." The observations don't include enough test coverage to determine which was intended.

### Missing context

The observations don't include the full `append` method body (only the first two lines at 141-143) or any `write_batch`/batch method. Seeing the actual call sites for `_do_sync(force=True)` would confirm the batch commit pattern and reveal whether there are other callers that use `force`.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:append` — See the full append method to confirm it calls `_do_sync()` without force and trace the complete write-then-sync sequence
- [function] `write-ahead-log/wal.py:_rotate` — Rotation does unconditional fsync — this is the safety net that eventually persists data even in `none` mode
- [general] `batch-write-durability` — Find the batch write method that uses `force=True` and verify whether it provides atomicity (all-or-nothing) or just durability (all-flushed)
- [file] `write-ahead-log/test_wal.py` — Check whether tests exercise the durability gap: what happens if a crash occurs mid-batch in `none` mode
- [general] `write-count-reset-semantics` — Determine whether `_write_count` should reset on any fsync (including forced) or only on batch-threshold syncs — potential bug

## Beliefs

- `batch-mode-defers-fsync` — In `batch` sync mode, individual appends do not fsync until `_write_count` reaches `_batch_sync_count` (default 100), leaving up to N-1 records vulnerable to crash loss
- `none-mode-never-syncs-on-append` — In `none` sync mode, `_do_sync` is a no-op for individual appends; data reaches disk only via rotation or explicit close
- `force-true-bypasses-sync-mode` — Passing `force=True` to `_do_sync` causes flush+fsync regardless of the configured sync mode, enabling group commit at batch boundaries
- `rotation-always-fsyncs` — The `_rotate` method unconditionally calls `flush()` and `os.fsync()` before closing the current file, providing a durability checkpoint even in `none` mode
- `batch-write-count-not-reset-on-forced-sync` — The `_write_count` counter only resets when the batch threshold triggers a sync, not when a forced sync occurs, which may cause counter drift

