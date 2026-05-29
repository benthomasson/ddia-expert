# Topic: Find the batch write method that uses `force=True` and verify whether it provides atomicity (all-or-nothing) or just durability (all-flushed)

**Date:** 2026-05-29
**Time:** 08:22

## Batch Write with `force=True`: Atomicity + Durability

The method in question is **`append_batch`** at `write-ahead-log/wal.py:155`. Here's what it does:

### The Mechanism

```python
def append_batch(self, operations: List[Tuple[str, str, str]]) -> int:
    """Atomically append a batch with COMMIT. Returns COMMIT sequence number."""
    with self._lock:
        buf = bytearray()
        for op_type, key, value in operations:
            self._seq_num += 1
            buf.extend(_encode_record(...))
        # Append a COMMIT sentinel
        self._seq_num += 1
        commit_seq = self._seq_num
        buf.extend(_encode_record(commit_seq, OP_COMMIT, b"", b""))
        # Single write, then forced fsync
        self._fd.write(bytes(buf))
        self._do_sync(force=True)
```

### It provides **both** atomicity and durability

**Durability** comes from `force=True`. Looking at `_do_sync` (`wal.py:124`), when `force=True`, it unconditionally calls `flush()` + `os.fsync()`, bypassing the normal sync mode logic (which might batch or skip syncs). This guarantees all bytes hit stable storage before `append_batch` returns.

**Atomicity** comes from the **COMMIT record protocol**, not from the physical write. The batch is structured as:

1. N operation records (PUT/DELETE)
2. One `OP_COMMIT` record as the final sentinel

During recovery, a reader scanning the WAL (as `_read_record` at `wal.py:40` does) will encounter one of three states:

| Crash timing | What recovery sees | Result |
|---|---|---|
| Before any write | No records | Batch discarded |
| Mid-write (partial records) | Truncated record → `_read_record` returns `None` on short read, or `ValueError` on CRC mismatch (`wal.py:56`) | Batch discarded — no COMMIT found |
| After all records but before COMMIT fsynced | Complete operation records but no COMMIT | Batch discarded |
| After fsync completes | All records + COMMIT | Batch applied |

The CRC per record (`zlib.crc32` at `wal.py:32-33`) catches torn writes — a partially written record will fail checksum validation.

### One caveat

The recovery code shown (`_recover_seq_num` at `wal.py:92`) only finds the max sequence number — it doesn't show the actual replay logic that would skip uncommitted batches. The COMMIT record type (`OP_COMMIT = 3`, `wal.py:12`) exists and is written, but the consumer-side logic that enforces "only apply records preceding a COMMIT" is **not visible in the observations**. The docstring says "Atomically append a batch with COMMIT," and the protocol is correct, but the atomicity guarantee is only as strong as the recovery consumer's discipline in checking for COMMIT.

### Why `force=True` matters specifically here

Compare with `append` at `wal.py:147`, which calls `self._do_sync()` **without** `force=True`. In `batch` sync mode, individual appends accumulate up to `batch_sync_count` before fsyncing (`wal.py:130-133`). But `append_batch` and `checkpoint` (`wal.py:170`) both force an immediate fsync — because a batch that returns without syncing could lose the COMMIT record on crash, breaking the atomicity guarantee.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_record` — The CRC validation and truncation handling that makes crash recovery safe; understanding this is key to trusting the atomicity claim
- [general] `wal-recovery-consumer` — Find the code that replays WAL records and verify it actually groups records by COMMIT boundaries rather than applying them individually
- [function] `write-ahead-log/wal.py:truncate` — How completed batches are garbage-collected after checkpoint, and whether truncation itself is crash-safe
- [file] `log-structured-merge-tree/lsm.py` — The LSM tree has its own simpler WAL class (line 13) without COMMIT records — compare the two WAL designs to understand when atomicity matters vs. when per-record durability suffices
- [function] `write-ahead-log/wal.py:_do_sync` — The three sync modes (sync/batch/none implied) and how `force=True` overrides them; important for understanding the durability-performance tradeoff

## Beliefs

- `wal-append-batch-uses-commit-sentinel` — `append_batch` writes all operation records plus a trailing `OP_COMMIT` record into a single buffer before writing, using the COMMIT as a logical atomicity boundary
- `wal-force-true-bypasses-batch-sync` — `_do_sync(force=True)` unconditionally calls `flush()` + `os.fsync()`, ignoring the configured `sync_mode` and `batch_sync_count` settings
- `wal-crc-per-record` — Every WAL record includes a CRC32 checksum over `op_type + key + value`, validated on read; a mismatch raises `ValueError` which halts replay
- `wal-append-batch-and-checkpoint-always-fsync` — Both `append_batch` and `checkpoint` call `_do_sync(force=True)`, while single `append` defers to the configured sync mode — these are the only two operations that guarantee immediate durability
- `wal-atomicity-depends-on-consumer` — The COMMIT record protocol provides the mechanism for atomicity, but the actual enforcement depends on recovery consumers correctly treating records without a trailing COMMIT as uncommitted — this consumer logic is not present in the WAL module itself

