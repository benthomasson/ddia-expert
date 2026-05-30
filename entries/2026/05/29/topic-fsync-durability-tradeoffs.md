# Topic: How the three sync modes (sync, batch, none) trade latency for durability, and what "acknowledged" really means in each mode

**Date:** 2026-05-29
**Time:** 12:28

# Sync Modes in the Write-Ahead Log: Trading Latency for Durability

## The Core Mechanism: `_do_sync`

Everything hinges on one method at `write-ahead-log/wal.py:125-133`:

```python
def _do_sync(self, force: bool = False):
    """Fsync based on sync mode."""
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

Notice what's *absent*: there's no `elif self._sync_mode == "none"` branch. If the mode is `"none"`, neither `flush()` nor `fsync()` is called — the data sits in Python's userspace buffer and the OS page cache, at the mercy of the kernel's writeback schedule and a clean process exit.

## The Three Modes

### `"sync"` — Every write is durable before return

Every call to `append()` (line 148-149) triggers `_do_sync()`, which calls `flush()` + `os.fsync()`. The `fsync` syscall blocks until the kernel confirms the data has reached stable storage (disk platters, flash cells — past the volatile write cache, assuming the drive honors fsync correctly).

**What "acknowledged" means:** When `append()` returns your sequence number, that record is on disk. A power failure one nanosecond later won't lose it. This is the strongest guarantee and the slowest mode — each write pays the full disk I/O latency (typically 0.1–10ms for SSD, much worse for spinning rust).

### `"batch"` — Amortize fsync across N writes

The WAL counts writes and only fsyncs every `batch_sync_count` operations (default: 100, set at line 66). Between fsyncs, data accumulates in the OS page cache.

**What "acknowledged" means:** When `append()` returns, the data has been written to the kernel buffer (via Python's file write) but may *not* be on disk yet. Up to `batch_sync_count - 1` acknowledged records could be lost on a crash. The 100th write triggers the fsync that makes all 100 durable at once. This is a classic throughput optimization — you trade a bounded window of potential data loss for dramatically higher write throughput.

### `"none"` — No explicit durability

No `flush()`, no `fsync()`. Data is written to Python's internal buffer, and eventually to the OS page cache when the buffer fills or on `close()`. The OS may write it to disk whenever it feels like it.

**What "acknowledged" means:** When `append()` returns, the data is in your process's memory. It is *not* guaranteed to be in the kernel page cache, let alone on disk. A process crash (segfault, OOM kill) can lose data that never made it past Python's buffer. A kernel crash or power failure can lose data that made it to the page cache but not to disk. This mode is only appropriate for data you can afford to lose — e.g., a performance log, or a WAL in front of a system that has other durability mechanisms.

## The Critical Exception: `force=True`

Both `append_batch()` (line 163) and `checkpoint()` (line 170) call `_do_sync(force=True)`, which **always** fsyncs regardless of mode. This means:

- **Batch commits are always durable**, even in `"batch"` or `"none"` mode. When `append_batch()` returns a commit sequence number, that entire atomic batch is on disk.
- **Checkpoints are always durable.** A checkpoint marks "everything up to here has been applied to the main data store" — losing that marker would cause redundant replay, so it's always forced to disk.

This is a deliberate design choice: the sync mode governs the durability of *individual* `append()` calls, but *transactional* operations (`append_batch`) and *recovery markers* (`checkpoint`) always get the full fsync treatment. The `"none"` and `"batch"` modes weaken single-record durability but preserve transactional durability.

## The Latency/Durability Spectrum

| Mode | `append()` durability | `append_batch()` durability | Latency per write | Risk window |
|------|----------------------|----------------------------|-------------------|-------------|
| `"sync"` | On disk | On disk | ~1 fsync | 0 records |
| `"batch"` | In page cache | On disk | ~1/100th fsync | Up to 99 records |
| `"none"` | In process buffer | On disk | ~0 (memcpy) | All unbatched records |

## Topics to Explore

- [function] `write-ahead-log/wal.py:append_batch` — How the COMMIT record makes batches atomic and why it always forces fsync, creating a "transaction boundary" even in relaxed sync modes
- [function] `write-ahead-log/wal.py:_rotate` — Rotation always fsyncs before closing the old file (line 114-115), which means file rotation is an implicit durability barrier regardless of sync mode
- [function] `write-ahead-log/wal.py:truncate` — How truncation interacts with durability: it fsyncs before rewriting files, ensuring the pre-truncation state is durable before destructive changes
- [file] `log-structured-merge-tree/lsm.py` — The LSM tree that sits *above* a WAL; understanding how it uses WAL durability to avoid fsyncing its own memtable helps explain why `"none"` mode can be safe in practice
- [general] `os-fsync-vs-fdatasync` — This implementation uses `os.fsync` everywhere; `fdatasync` skips metadata updates and is faster on Linux — worth understanding when that optimization is safe

## Beliefs

- `wal-batch-and-checkpoint-always-fsync` — `append_batch()` and `checkpoint()` call `_do_sync(force=True)`, guaranteeing disk durability regardless of the configured sync mode
- `wal-none-mode-skips-all-sync` — In `"none"` mode, individual `append()` calls perform no `flush()` or `fsync()`; data remains in Python's userspace buffer until the OS or a forced sync flushes it
- `wal-batch-mode-counter-resets-on-sync` — Batch mode counts writes via `_write_count` and fsyncs every `batch_sync_count` (default 100) operations, resetting the counter to zero after each fsync
- `wal-rotation-forces-fsync` — `_rotate()` always calls `flush()` + `os.fsync()` on the outgoing file before closing it, so file rotation is a durability barrier in all sync modes

