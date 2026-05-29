# Topic: Building a correct replay that groups records by batch and checks for COMMIT before applying, using `iterate()` as the foundation

**Date:** 2026-05-28
**Time:** 18:51

The observations cut off at line 200 of `wal.py` (total 264 lines), so I'm missing the `replay()`, `iterate()`, `close()`, and `current_seq_num()` methods. But the test suite reveals exactly how they behave, which is enough to explain the design.

---

# Building Correct Replay from `iterate()`

## The two-layer design

The WAL has two read APIs with different abstraction levels:

**`iterate()`** is the raw record stream. It yields every record in sequence order — including `COMMIT` and `CHECKPOINT` markers. From `test_iterate` (test_wal.py:89-95):

```python
wal.append("PUT", "x", "1")
wal.append_batch([("PUT", "y", "2"), ("PUT", "z", "3")])
all_recs = list(wal.iterate())
assert len(all_recs) == 4          # x, y, z, COMMIT
assert all_recs[3].op_type == "COMMIT"
```

**`replay()`** is the crash-safe view. It returns only data-bearing records (`PUT`/`DELETE`) from committed batches, filtering out `COMMIT` and `CHECKPOINT` markers. From `test_basic` (test_wal.py:23-24):

```python
records = wal.replay()
assert len(records) == 6    # 3 individual + 3 batch ops; no COMMIT, no CHECKPOINT
```

That count — 6, not 8 — is the key. Eight records were written (3 individual + 3 batch + 1 COMMIT + 1 CHECKPOINT), but replay returns only the six data operations.

## Why batching needs COMMIT checks

Look at how `append_batch` works (wal.py lines ~153–168). It writes all operations and the `COMMIT` record into a single buffer, then flushes atomically:

```python
def append_batch(self, operations):
    with self._lock:
        buf = bytearray()
        for op_type, key, value in operations:
            self._seq_num += 1
            buf.extend(_encode_record(...))
        self._seq_num += 1
        commit_seq = self._seq_num
        buf.extend(_encode_record(commit_seq, OP_COMMIT, b"", b""))
        self._fd.write(bytes(buf))
        self._do_sync(force=True)
```

The `force=True` on sync is deliberate — batch boundaries are always fsynced regardless of sync mode. But a crash could still occur mid-write, leaving partial batch records on disk without the trailing `COMMIT`. A correct `replay()` must handle this.

## How replay groups by batch

Though I can't see the implementation directly, the tests constrain exactly one correct algorithm:

1. **Call `iterate()`** to get the raw record stream (including `COMMIT`/`CHECKPOINT`)
2. **Accumulate a pending batch** — records that follow a previous COMMIT (or the start) but haven't yet seen their own COMMIT
3. **On seeing `COMMIT`**: flush the pending batch into the results list
4. **On seeing standalone records** (those written via `append()`, not `append_batch()`): add directly to results — they were individually fsynced and don't need a COMMIT gate
5. **Skip `CHECKPOINT` records** — they're metadata, not data
6. **At EOF, discard any pending uncommitted batch** — this is the crash recovery case

The distinction between standalone and batched records is the crux. Individual `append()` calls (wal.py:140-149) fsync each record independently, so they're durable the moment `append()` returns. Batched records only become durable when the `COMMIT` is fsynced.

## The crash recovery invariant

`test_corruption` (test_wal.py:48-59) demonstrates the safety property: when trailing bytes are corrupted, replay stops and returns only intact records. Since `_read_record` (wal.py:40-62) validates CRC on every record, a partial batch from a crash would either:
- Fail CRC check on a partially-written record → replay stops before seeing COMMIT
- Be complete up to some record but missing the COMMIT → replay discards the uncommitted group

Either way, partially-written batches never appear in replay output. The `after_seq` parameter (tested in tester_test_wal.py:137-149) layers filtering on top — only returning records past a checkpoint boundary, which is how you avoid re-applying already-applied operations.

## Topics to Explore

- [function] `write-ahead-log/wal.py:replay` — The actual implementation of batch grouping and COMMIT checking (lines 200-264, not captured in observations)
- [function] `write-ahead-log/wal.py:iterate` — The raw record iterator that replay builds on — understanding its EOF and error handling is critical
- [general] `partial-write-detection` — How CRC32 in `_read_record` (line 57-59) interacts with OS write atomicity to detect torn writes
- [function] `write-ahead-log/wal.py:truncate` — How truncation interacts with batch boundaries — can you truncate mid-batch?
- [general] `fsync-semantics-by-mode` — The `_do_sync` method (lines 126-134) treats batches specially with `force=True` — explore what happens to individual appends under `batch` or `none` sync modes

## Beliefs

- `replay-excludes-commit-and-checkpoint` — `replay()` returns only PUT and DELETE records; COMMIT and CHECKPOINT markers are filtered out, as evidenced by test_basic expecting 6 records from 8 written
- `append-batch-always-force-syncs` — `append_batch()` calls `_do_sync(force=True)`, bypassing the sync mode setting to ensure batch atomicity regardless of whether sync mode is "sync", "batch", or "none"
- `iterate-yields-all-record-types` — `iterate()` yields every record including COMMIT and CHECKPOINT, making it the raw primitive that replay is built on
- `uncommitted-batch-records-discarded-on-replay` — If a crash occurs mid-batch (after some ops but before COMMIT is written), replay must discard those orphaned records to maintain atomicity
- `crc-validates-op-key-value-not-header` — CRC32 in `_encode_record` covers `op_type_byte + key + value` (line 31) but not the length/seq_num header fields, meaning header corruption could cause misframing rather than a clean CRC error

