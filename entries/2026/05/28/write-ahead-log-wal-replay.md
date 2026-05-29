# Function: replay in write-ahead-log/wal.py

**Date:** 2026-05-28
**Time:** 19:21

# `WriteAheadLog.replay`

## Purpose

`replay` reconstructs the state of committed operations after a crash or restart. It reads the WAL files on disk and returns all `PUT` and `DELETE` records whose sequence numbers are newer than a given checkpoint. This is the core recovery mechanism — a consumer calls `replay` after a crash to re-apply operations that were logged but may not have been flushed to the primary data store.

## Contract

**Preconditions:**
- `after_seq` must be a non-negative integer. The method doesn't validate this — a negative value would effectively mean "return everything," which happens to work but isn't intentional.
- WAL files in `self._dir` must not be externally modified between the flush and the read. There's no file-level lock protecting against concurrent external writers.

**Postconditions:**
- Returns records in log order (file order, then byte offset within file).
- Every returned record has `op_type` of either `"PUT"` or `"DELETE"`.
- Every returned record has `seq_num > after_seq`.
- CRC integrity has been verified for every returned record (enforced by `_read_record`).

**Invariant:** The returned list is a subset of what `iterate()` would return, filtered to data-bearing operations above the sequence threshold.

## Parameters

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `after_seq` | `int` | `0` | Sequence number high-water mark. Only records **strictly greater** than this are returned. Pass `0` to replay the entire log; pass the sequence number from your last checkpoint to get only what's new. |

## Return Value

`List[WALRecord]` — an in-memory list of all qualifying records. The caller gets a snapshot, not a lazy iterator, so this can be large if the WAL is large. Each `WALRecord` contains `seq_num`, `op_type`, `key`, `value`, and `checksum`.

## Algorithm

1. **Flush buffered writes.** Acquires `self._lock` and flushes the file descriptor. This ensures any `append` calls that returned before `replay` was called are visible on disk. The lock is released immediately — the read phase is lock-free.

2. **Scan all WAL files.** Delegates to `_read_all_records()`, which iterates WAL files in sorted filename order, reading records sequentially from each. If any record fails CRC validation, `_read_all_records` **stops entirely** (returns, not continues) — corruption truncates the replay at that point.

3. **Filter by sequence number.** Records with `seq_num <= after_seq` are skipped.

4. **Filter by operation type.** Only `PUT` and `DELETE` records pass through. `COMMIT` and `CHECKPOINT` records are control markers and are discarded.

## Side Effects

- **I/O:** Flushes the current write file descriptor (one `flush()` call). Then opens and reads every `.wal` file in the log directory.
- **No state mutation:** Does not modify `self._seq_num`, the WAL files, or any other instance state beyond the flush.

## Error Handling

- **CRC mismatch:** `_read_record` raises `ValueError` on checksum failure. `_read_all_records` catches this and **stops iteration** — any records after corruption are silently lost. This is a deliberate design choice: corruption means the write was incomplete (likely a crash mid-write), so everything from that point forward is suspect.
- **Truncated records:** If a record is partially written (short read), `_read_record` returns `None`, which ends iteration for that file. `_read_all_records` then moves to the next file.
- **Missing directory / permission errors:** Not caught here — will propagate as `OSError`.

## Usage Patterns

Typical crash-recovery flow:

```python
wal = WriteAheadLog("/var/data/wal")
last_checkpoint_seq = load_checkpoint_from_store()
records = wal.replay(after_seq=last_checkpoint_seq)
for rec in records:
    if rec.op_type == "PUT":
        store.put(rec.key, rec.value)
    elif rec.op_type == "DELETE":
        store.delete(rec.key)
store.flush()
wal.checkpoint()
```

The caller is responsible for applying the records idempotently — `replay` may return the same records across multiple calls if no new checkpoint or truncation occurs between them.

## Dependencies

- **`_read_all_records`** — the internal iterator that does the actual file I/O and corruption handling.
- **`_read_record`** (module-level) — binary deserialization and CRC verification of individual records.
- **`_wal_files`** — discovers and sorts WAL segment files by filename.
- **`threading.Lock`** — protects the flush but not the read phase.

## Notable Design Decision

The docstring says "skips uncommitted batches," but the implementation doesn't actually track batch boundaries. There's no batch-start marker in the format, so `replay` cannot distinguish a batch's PUT/DELETE records from individual writes. It returns **all** PUT/DELETE records regardless of whether a matching COMMIT exists. The inline comment acknowledges this explicitly. In practice, this means a crash mid-batch will replay the partial batch — the atomicity guarantee of `append_batch` only holds at the fsync level (all-or-nothing write to disk), not at the replay-filtering level.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_all_records` — The corruption-stopping iterator that `replay` delegates to; understanding its early-termination behavior is essential to reasoning about data loss scenarios
- [function] `write-ahead-log/wal.py:append_batch` — How batch atomicity is achieved via a trailing COMMIT record and forced fsync, and how that interacts with replay's filtering
- [function] `write-ahead-log/wal.py:truncate` — The complementary operation that removes already-applied records; understanding the truncate-replay lifecycle is key to WAL-based recovery
- [file] `log-structured-merge-tree/test_lsm.py` — Shows how the LSM tree uses WAL replay during crash recovery, the primary consumer of this method
- [general] `wal-batch-atomicity-gap` — The gap between the docstring's claim ("skips uncommitted batches") and the implementation (returns all PUT/DELETE regardless of COMMIT presence)

## Beliefs

- `replay-returns-only-put-delete` — `replay` filters out COMMIT and CHECKPOINT records; only PUT and DELETE operations appear in the returned list
- `replay-stops-at-corruption` — If any record fails CRC validation, all subsequent records across all remaining WAL files are silently dropped from the replay result
- `replay-does-not-enforce-batch-atomicity` — Despite the docstring, replay does not track batch boundaries and will return partial batch records if a crash occurred mid-batch before the COMMIT was written
- `replay-flush-before-read` — `replay` flushes the write file descriptor under lock before reading, ensuring all prior `append` calls are visible on disk
- `replay-is-not-lock-protected-during-read` — The file read phase runs without holding `self._lock`, so concurrent `append` calls during replay could produce a partial read of in-flight writes

