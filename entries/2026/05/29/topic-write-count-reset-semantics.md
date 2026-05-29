# Topic: Determine whether `_write_count` should reset on any fsync (including forced) or only on batch-threshold syncs — potential bug

**Date:** 2026-05-29
**Time:** 08:23

# `_write_count` Reset Behavior — Potential Bug in Batch Sync Mode

## The Bug

In `write-ahead-log/wal.py:124-134`, the `_do_sync` method has a subtle logic error when `sync_mode == "batch"` and `force=True`:

```python
def _do_sync(self, force: bool = False):
    if self._sync_mode == "sync" or force:          # line 126
        self._fd.flush()
        os.fsync(self._fd.fileno())
    elif self._sync_mode == "batch":                 # line 129
        self._write_count += 1
        if self._write_count >= self._batch_sync_count:
            self._fd.flush()
            os.fsync(self._fd.fileno())
            self._write_count = 0                    # line 134
```

The `if`/`elif` structure means the two branches are **mutually exclusive**. When `sync_mode == "batch"` and `force=True`, the first branch fires (because `force` is true), performs the fsync, but **never touches `_write_count`**. The `elif` branch — the only place where `_write_count` is incremented or reset — is skipped entirely.

## Where Forced Syncs Happen

Two callers pass `force=True`:

- **`append_batch`** (`wal.py:165`) — atomic batch writes with a COMMIT record must be durable immediately
- **`checkpoint`** (`wal.py:175`) — checkpoint records must also be durable

Regular `append` calls (`wal.py:149`) use the default `force=False`, so in batch mode they go through the `elif` branch and correctly manage the counter.

## Concrete Scenario

With `batch_sync_count=100`:

1. 50 individual `append` calls → `_write_count` reaches 50
2. `append_batch` fires → forced fsync flushes **everything** to disk, but `_write_count` stays at 50
3. 50 more `append` calls → `_write_count` reaches 100, triggers a batch sync

That batch sync at step 3 fires after only 50 unsynced writes, not 100. The 50 writes from step 1 were already durable after the forced fsync in step 2, but they're still counted toward the threshold.

## Impact Assessment

**This is a performance bug, not a correctness bug.** Data is never lost — the forced sync guarantees durability, and the premature batch sync just adds an unnecessary extra fsync. In workloads that mix `append` with `append_batch` or `checkpoint`, batch mode will sync more frequently than the configured threshold intends.

The severity depends on workload shape. If forced syncs are rare relative to `batch_sync_count`, the effect is negligible. If they're frequent (e.g., many small batches), the counter almost never reaches the threshold organically — it keeps getting "frozen" by forced syncs and then slowly climbing from a stale base.

## The Fix

Reset `_write_count` in the forced-sync path when in batch mode:

```python
def _do_sync(self, force: bool = False):
    if self._sync_mode == "sync" or force:
        self._fd.flush()
        os.fsync(self._fd.fileno())
        if force and self._sync_mode == "batch":
            self._write_count = 0
    elif self._sync_mode == "batch":
        self._write_count += 1
        if self._write_count >= self._batch_sync_count:
            self._fd.flush()
            os.fsync(self._fd.fileno())
            self._write_count = 0
```

This ensures the counter reflects reality: after any fsync, there are zero unsynced writes pending.

## Contrast with Other Implementations

For comparison, `hash-index-storage/bitcask.py:88` takes a simpler approach — it syncs on every write when `sync_writes=True`, with no batching mode at all. The LSM tree's WAL (`log-structured-merge-tree/lsm.py:27`) also does a `flush()` per write with no fsync batching. The batch-sync logic is unique to the WAL implementation, which is why this counter-management bug only surfaces here.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_do_sync` — Trace all callers to verify no other paths bypass the counter
- [function] `write-ahead-log/wal.py:_rotate` — Rotation does its own fsync at line 115 without going through `_do_sync`, so `_write_count` is also not reset on rotation
- [file] `write-ahead-log/test_wal.py` — Check whether tests cover batch-mode behavior after forced syncs (line 106 tests all three sync modes but may not exercise the interleaving)
- [function] `write-ahead-log/wal.py:truncate` — Another fsync at line 184 that bypasses `_do_sync` and doesn't reset the counter
- [general] `sync-mode-none-safety` — When `sync_mode="none"`, `_do_sync` with `force=False` does nothing — verify callers that need durability always pass `force=True`

## Beliefs

- `wal-write-count-not-reset-on-forced-sync` — In batch sync mode, `_write_count` is not reset when a forced fsync occurs via `append_batch` or `checkpoint`, causing the next batch-threshold sync to fire prematurely
- `wal-do-sync-branches-mutually-exclusive` — The `_do_sync` method's `if`/`elif` structure means forced syncs in batch mode skip all counter logic — the counter is neither incremented nor reset
- `wal-rotate-bypasses-do-sync` — File rotation at `wal.py:115` calls `os.fsync` directly without going through `_do_sync`, so `_write_count` is not reset on rotation either
- `wal-batch-bug-is-performance-not-correctness` — The stale `_write_count` causes extra fsyncs (premature batch threshold) but never causes data loss, since forced syncs always flush to disk regardless of the counter

