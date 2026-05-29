# Topic: The `sync_mode` parameter and `batch_sync_count` determine how often `fsync` is called, which directly controls the window of vulnerability to torn writes

**Date:** 2026-05-29
**Time:** 08:18

# How `sync_mode` and `batch_sync_count` Control the Torn Write Window

## The Core Mechanism: `_do_sync`

The entire durability story lives in one method — `_do_sync` at `write-ahead-log/wal.py:125`:

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

There are three modes, and each defines a different durability contract:

### Mode 1: `"sync"` (default)

Every single `append()` call triggers `flush()` + `os.fsync()` (lines 127–128). The write hits stable storage before `append` returns. The vulnerability window is essentially zero — if the process crashes after `append` returns, the record is on disk. This is the safest mode and the default (`wal.py:64`: `sync_mode: str = "sync"`).

### Mode 2: `"batch"`

Writes accumulate in an OS buffer. The WAL tracks a counter (`_write_count`, initialized at line 69) and only calls `fsync` when the counter reaches `_batch_sync_count` (line 131: `if self._write_count >= self._batch_sync_count`). The default batch size is 100 (line 66). This means up to 99 records can sit in OS buffers without being fsync'd — if the machine loses power during that window, those records are lost or partially written (torn).

The tradeoff is throughput: batching amortizes the cost of `fsync` (which can take milliseconds on spinning disks) across many writes. But the vulnerability window grows linearly with `batch_sync_count`.

### Mode 3: `"none"` (implicit)

If `sync_mode` is neither `"sync"` nor `"batch"`, `_do_sync` does nothing — no `flush()`, no `fsync()`. The data sits in userspace and OS buffers indefinitely. The vulnerability window extends until the OS decides to write dirty pages back, or until something else forces a sync. This is the fastest and the most dangerous.

## Where `force=True` Bypasses the Mode

Two operations always fsync regardless of mode:

- **`append_batch`** (line 170): `self._do_sync(force=True)` — Batch commits are atomic boundaries. A half-written batch without the COMMIT record is useless (replay skips uncommitted batches, per line 215), so the COMMIT must reach disk.
- **`checkpoint`** (line 183): `self._do_sync(force=True)` — Checkpoints mark a known-good recovery point. If they aren't durable, the entire truncation/recovery protocol breaks.

This is a deliberate design: individual puts can tolerate batched durability, but transactional boundaries (COMMIT, CHECKPOINT) cannot.

## How CRC Detects Torn Writes After the Fact

The `sync_mode` controls *prevention* — how much data is at risk. CRC32 checksums handle *detection*. Each record's checksum covers the op type, key, and value (line 30: `crc_data = struct.pack("B", op_type_byte) + key + value`). On read, `_read_record` recomputes the CRC and raises `ValueError` on mismatch (lines 52–54). The test at `test_wal.py:44` (`test_corruption`) verifies this: it overwrites the last 5 bytes of a WAL file and confirms that replay recovers only the intact first record.

A partial/torn write produces either a short read (caught at lines 40–41 and 44–45, returning `None`) or a CRC mismatch. Either way, replay stops at the corruption boundary — it doesn't silently return bad data.

## The Durability Spectrum

| Mode | fsync frequency | Records at risk | Use case |
|------|----------------|-----------------|----------|
| `"sync"` | Every write | 0 | Financial transactions, small write volume |
| `"batch"` (n=100) | Every 100 writes | Up to 99 | High-throughput logging, acceptable loss window |
| `"none"` | Never (by WAL) | All unbuffered | Testing, ephemeral data |

The `batch_sync_count` parameter lets you tune the batch window precisely — set it to 10 for a tighter window with moderate throughput, or 1000 for maximum throughput with a wider risk window.

## Topics to Explore

- [function] `write-ahead-log/wal.py:append_batch` — Shows how transactional atomicity is achieved by writing all ops + COMMIT in a single buffer, then force-syncing — the only path that always bypasses `sync_mode`
- [function] `write-ahead-log/wal.py:_read_record` — The deserialization and CRC validation logic that actually detects torn writes at recovery time
- [function] `write-ahead-log/wal.py:truncate` — How completed records are removed from WAL files; interacts with fsync to ensure the rewrite is durable before deleting originals
- [file] `write-ahead-log/wal.py` (lines 210–260) — The `replay` and `read_all` methods that implement recovery semantics: skipping uncommitted batches and stopping at corruption
- [general] `fsync-vs-fdatasync` — This implementation uses `os.fsync` which also syncs file metadata; `fdatasync` would be faster on Linux by skipping metadata — worth understanding the tradeoff

## Beliefs

- `wal-sync-mode-default` — `WriteAheadLog` defaults to `sync_mode="sync"`, calling `flush()` + `os.fsync()` after every single `append` call
- `wal-batch-force-sync` — `append_batch` and `checkpoint` always force an fsync regardless of `sync_mode`, ensuring transactional boundaries are durable
- `wal-batch-counter-reset` — In batch mode, the write counter resets to 0 after each fsync, meaning the vulnerability window is always between 0 and `batch_sync_count - 1` records
- `wal-none-mode-no-sync` — When `sync_mode` is `"none"`, `_do_sync` performs no I/O at all — durability depends entirely on OS page cache writeback
- `wal-crc-detects-torn-writes` — CRC32 checksums cover op type + key + value, and `_read_record` raises `ValueError` on mismatch, which causes replay to stop at the corruption point rather than return corrupt data

