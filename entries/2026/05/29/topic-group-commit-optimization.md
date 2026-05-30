# Topic: The batch mode here is a simplified version of the group commit technique used in real databases (PostgreSQL, MySQL) to amortize fsync cost across multiple transactions

**Date:** 2026-05-29
**Time:** 12:22

## Batch Sync Mode as Simplified Group Commit

The WAL implementation in `write-ahead-log/wal.py` supports three `sync_mode` settings: `"sync"`, `"batch"`, and `"none"`. The batch mode is the one that mirrors the group commit pattern from production databases.

### How it works in this codebase

The key method is `_do_sync` at line ~124 of `wal.py`:

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

In `"sync"` mode, every single `append()` call (line ~141) triggers an `os.fsync` — the safest but slowest option. An fsync forces the OS to flush data from kernel buffers to physical disk, which typically takes 5–10ms on spinning disks and 0.1–1ms on SSDs. At one fsync per write, throughput is capped at ~100–200 writes/sec on HDD.

In `"batch"` mode, the WAL increments `_write_count` and only fsyncs when the counter hits `_batch_sync_count` (default 100, set at line 66). So 100 writes share a single fsync call. That's the amortization: the same ~10ms cost is divided across 100 operations instead of being paid individually.

Note that `append_batch()` (line 153) always passes `force=True` to `_do_sync`, meaning batch transactions always get an immediate fsync regardless of mode. This is correct — a COMMIT record must be durable before the caller can consider the transaction committed.

### What real group commit does differently

In PostgreSQL's group commit (and MySQL's `binlog_group_commit_sync_delay`), the mechanism is fundamentally **concurrent**:

1. Multiple transactions finish their work and queue up waiting for an fsync.
2. A "leader" transaction collects all pending commit requests.
3. The leader does a single fsync that covers all of them.
4. The leader wakes up all the waiting transactions: "your data is durable."

The critical difference is that real group commit coordinates **concurrent waiters**. There's a queue, a leader election, and a wake-up notification. The throughput gain comes from the fact that while one transaction is waiting for the disk, other transactions pile up behind it — and they all get covered by the next fsync for free.

This implementation simplifies that to a **counter** (`_write_count`). There are no concurrent waiters. The 99 writes between fsyncs simply aren't durable until the 100th write triggers the flush. If the process crashes after write 50, all 50 are potentially lost. In real group commit, each waiting transaction gets a guarantee: once the leader's fsync completes, *your* data is safe.

The single `threading.Lock` at line ~72 serializes all writes, so there's no opportunity for concurrent transactions to naturally batch behind a shared fsync anyway.

### The tradeoff in `"batch"` mode

The `"batch"` mode trades durability for throughput. This is visible in `test_sync_modes` (`test_wal.py`, line 105), which just verifies all three modes can open/write/close without error — but does **not** test crash recovery under batch mode, because the semantics are weaker: some recent writes may be lost.

By contrast, `"sync"` mode gives you what databases call **write-ahead guarantee**: if `append()` returns, that record is on disk. The test at `test_crash_recovery` (line 29) relies on this — it reopens the WAL without calling `close()` and expects all records to be present.

## Topics to Explore

- [function] `write-ahead-log/wal.py:append_batch` — How atomic multi-record batches use COMMIT markers and forced fsync to guarantee all-or-nothing durability, distinct from the batch sync mode
- [function] `write-ahead-log/wal.py:_rotate` — File rotation triggers an fsync before closing, which interacts with batch mode by flushing any un-synced writes early
- [general] `postgresql-group-commit` — Study PostgreSQL's `CommitDelay` and `CommitSiblings` settings to see how real group commit coordinates concurrent waiters with a sleep window
- [file] `log-structured-merge-tree/lsm.py` — The LSM WAL (line 13–63) takes a different approach: it calls `flush()` on every append but never calls `fsync`, relying on the OS page cache — compare the durability tradeoffs
- [function] `event-sourcing-store/event_store.py:append_batch` — A higher-level batch append that does no fsync at all (plain file write with `json.dumps`), showing how application-level batching can exist independently of disk sync strategy

## Beliefs

- `wal-batch-mode-loses-up-to-N-writes` — In `"batch"` sync mode, up to `batch_sync_count - 1` writes (default 99) can be lost on crash because fsync only fires every Nth write
- `append-batch-always-fsyncs` — `append_batch()` passes `force=True` to `_do_sync`, so batched transactions always get an immediate fsync regardless of the configured sync mode
- `wal-no-concurrent-group-commit` — The WAL uses a single `threading.Lock` and a write counter rather than a concurrent waiter queue, so it cannot naturally amortize fsync across concurrent callers the way PostgreSQL's group commit does
- `lsm-wal-never-fsyncs` — The LSM tree's internal WAL (`lsm.py` line 26) calls `flush()` but never `os.fsync()`, offering weaker durability guarantees than the standalone WAL module's `"sync"` mode

