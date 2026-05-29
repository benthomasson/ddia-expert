# Function: checkpoint in write-ahead-log/wal.py

**Date:** 2026-05-29
**Time:** 08:02

## `WriteAheadLog.checkpoint()`

### Purpose

`checkpoint` writes a special marker record into the WAL that says "all data up to this point has been flushed to the primary storage." It exists so that during recovery, the system knows which WAL records can be safely skipped — anything before the most recent checkpoint is already durable in the main data store. This is the classic WAL checkpoint pattern from DDIA Chapter 3: the WAL grows unboundedly without checkpoints, and truncation needs a safe cutoff point.

### Contract

- **Precondition**: The WAL must be open (i.e., `self._fd` is not `None`). There's no guard against calling this on a closed WAL — it will raise `AttributeError`.
- **Postcondition**: A `CHECKPOINT` record with a new, unique sequence number is durably written to disk. The caller can later pass this sequence number to `truncate()` to discard everything at or before it.
- **Invariant**: Sequence numbers are strictly monotonically increasing under the lock, so the checkpoint's sequence number is greater than all prior records.

### Parameters

None. The method takes no arguments — it unconditionally records "now" as a checkpoint boundary.

### Return Value

Returns `int` — the sequence number assigned to this checkpoint record. The caller is expected to store this somewhere (e.g., alongside the flushed state of the primary storage) so it can later be passed to `truncate(up_to_seq)` or `replay(after_seq)`.

### Algorithm

1. **Acquire the lock** — serializes against concurrent `append`, `append_batch`, `truncate`, and other `checkpoint` calls.
2. **Increment `_seq_num`** — claims the next sequence number. This is the same counter used by all record types, so checkpoint records participate in the global total order.
3. **Encode and write** — calls `_encode_record` with `OP_CHECKPOINT`, empty key and value. The record still gets a CRC, length header, and full binary framing — same format as any other record, just with zero-length payloads.
4. **Force sync** — calls `_do_sync(force=True)`, which unconditionally flushes the userspace buffer and calls `os.fsync()`. The `force=True` bypasses the batch-sync counter — a checkpoint *must* be durable before the caller acts on it.
5. **Maybe rotate** — if the current WAL file now exceeds `_max_file_size`, opens a new segment file. This happens after the sync, so the checkpoint is guaranteed durable before rotation.
6. **Return the sequence number** — the caller uses this as the "safe to discard up to" marker.

### Side Effects

- **Disk I/O**: Writes bytes to the current WAL file and calls `fsync`. This is the expensive part — `fsync` blocks until the kernel confirms the data is on stable storage.
- **State mutation**: Increments `_seq_num` (permanently; there's no rollback).
- **Possible file rotation**: May close the current file and open a new segment, changing `_current_file` and `_fd`.

### Error Handling

There is no explicit error handling. If `_fd.write()` or `os.fsync()` raises an `OSError` (disk full, I/O error), it propagates to the caller while the lock is held — the `with self._lock` block releases the lock on exception unwind, but `_seq_num` has already been incremented, leaving a gap in the sequence. This is safe (gaps don't break replay) but worth knowing.

### Usage Patterns

Typical usage pairs `checkpoint` with `truncate`:

```python
# After flushing memtable/SSTable to disk:
seq = wal.checkpoint()
# Persist `seq` alongside the flushed state
wal.truncate(seq)  # discard everything up to and including the checkpoint
```

The caller is responsible for ensuring the primary storage is actually durable *before* calling `checkpoint` — otherwise recovery could skip records that weren't actually persisted. The WAL itself doesn't enforce this ordering; it's a protocol obligation.

### Dependencies

- `_encode_record` — shared binary encoder for all record types
- `_do_sync` — abstraction over sync modes (`sync`, `batch`, `none`)
- `_maybe_rotate` — file segment management
- `threading.Lock` — concurrency control
- `os.fsync` — kernel-level durability guarantee

The method notably does *not* interact with `replay()` or `truncate()` directly — the coordination happens through the sequence number returned to the caller.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:truncate` — The counterpart to checkpoint: how WAL segments are pruned using the sequence number checkpoint returns
- [function] `write-ahead-log/wal.py:replay` — How recovery uses sequence numbers to skip already-applied records, and why CHECKPOINT records are filtered out during replay
- [function] `write-ahead-log/wal.py:append_batch` — Compare the atomicity guarantee of batched writes (COMMIT marker) with the durability guarantee of checkpoint (force sync)
- [file] `write-ahead-log/test_wal.py` — Test cases reveal the expected checkpoint-truncate-replay protocol and edge cases around crash recovery
- [general] `wal-checkpoint-protocol` — DDIA's discussion of how checkpoints bound recovery time and enable log truncation (Chapter 3, SSTables and LSM-Trees)

---

## Beliefs

- `checkpoint-force-syncs` — `checkpoint()` always calls `fsync` regardless of the WAL's `sync_mode` setting, because checkpoint durability is a correctness requirement not a performance knob
- `checkpoint-shares-sequence-space` — Checkpoint records consume sequence numbers from the same monotonic counter as PUT, DELETE, and COMMIT records, maintaining a single total order across all operations
- `checkpoint-empty-payload` — Checkpoint records carry zero-length key and value fields; the record's significance is entirely in its op-type and sequence number
- `checkpoint-filtered-from-replay` — `replay()` only returns PUT and DELETE records, so checkpoint records are never visible to recovery consumers — they serve as truncation boundaries, not data

