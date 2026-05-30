# Topic: Designing a correct replay that groups records by batch using sequence number ranges between COMMIT markers, handling edge cases like interleaved individual writes and nested batches

**Date:** 2026-05-29
**Time:** 11:00

I can see from the observations that the file is 264 lines but only the first 200 were returned, cutting off the `replay` method. Let me work with what's available — the grep results give us the key lines.

---

# WAL Replay: Batch Grouping via COMMIT Markers

## The Core Problem

When a WAL is replayed after a crash, you must distinguish between **committed batches** (safe to replay) and **incomplete batches** (crash happened mid-write, must be discarded). The challenge is that the log is a flat stream of records — PUT, DELETE, COMMIT, CHECKPOINT — with no explicit "batch start" marker. The only grouping signal is the COMMIT record at the end.

## How This Implementation Handles It

### Writing: `append_batch` (wal.py:153-167)

The write side is straightforward. `append_batch` takes a list of operations, assigns each a consecutive sequence number, then appends a COMMIT record with its own sequence number at the end:

```
seq 4: PUT order:1 item_a
seq 5: PUT order:2 item_b
seq 6: DELETE order:0
seq 7: COMMIT          ← batch boundary
```

All records are serialized into a single `bytearray` buffer and written in one `self._fd.write(bytes(buf))` call (line 164), followed by a forced `fsync` (line 165). This makes the batch atomic at the filesystem level — either all bytes land on disk or none do (modulo sector-tearing edge cases).

### Reading: `replay` (wal.py:212-onwards)

The grep results reveal a critical design comment at lines 220-222:

> *Individual writes (no COMMIT needed) and batch writes (have COMMIT) are indistinguishable in the log format since there's no batch-start marker. All PUT/DELETE records are included; COMMIT/CHECKPOINT are [filtered out].*

This tells us the **actual replay strategy is flat** — it does **not** group records by batch. Instead, `replay` returns every PUT/DELETE record and simply skips COMMIT/CHECKPOINT markers. The method signature at line 212 shows it accepts an `after_seq` parameter for filtering by checkpoint.

### Why Flat Replay Works Here

This design sidesteps the batch-grouping problem entirely by relying on a simpler invariant: **if a COMMIT record is present, all preceding batch records are guaranteed to be on disk** (because they were written as a single buffer before the COMMIT). If the process crashed mid-batch, the final records would be truncated — `_read_record` returns `None` on partial reads (lines 43-44) and raises `ValueError` on CRC failures (line 55). The comment at line 215 confirms: replay "skips uncommitted batches; stops at corruption."

So the grouping logic is implicit:
1. Scan forward through all records
2. If a record is truncated or corrupt, **stop** — anything after corruption is unreliable
3. Return all valid PUT/DELETE records encountered before that point

This works because `append_batch` writes atomically. An incomplete batch means the COMMIT wasn't written, which means the final records before the crash will have a truncated read or CRC mismatch, triggering the stop.

### The Edge Case: Interleaved Individual Writes

Consider this sequence:

```
seq 1: PUT user:1 alice     ← individual write
seq 2: PUT user:2 bob       ← individual write
seq 3: PUT order:1 item_a   ← start of batch
seq 4: PUT order:2 item_b   ← part of batch
seq 5: COMMIT               ← batch end
```

The `test_basic` test (test_wal.py:6-26) exercises exactly this pattern: three individual appends followed by a three-operation batch. After replay, it asserts `len(records) == 6` — all six PUT/DELETE records, with COMMIT and CHECKPOINT filtered out.

Individual writes don't need COMMIT markers because each `append` call (wal.py:140-150) writes and syncs a single record independently. If a crash happens after seq 2 but before seq 3, replay recovers records 1 and 2 — both are complete, individually synced records.

### What About Nested Batches?

**They don't exist in this implementation.** There is no concept of nesting — `append_batch` holds the lock (line 155) for the entire batch, preventing any interleaving. A "nested batch" would require a separate OP type like `OP_BATCH_START` to delimit groups, which this format explicitly avoids (lines 221-222).

### The `iterate` vs `replay` Distinction

The `test_iterate` test (test_wal.py:86-93) reveals a second access pattern: `iterate()` yields **all** raw records including COMMIT markers. This is useful for tools that need to inspect the log structure itself, while `replay()` gives you the application-level view of "what state changes need to be applied."

## The Design Trade-off

This flat-replay approach is simple and correct for single-writer scenarios. The cost is that you lose batch grouping information during replay — the consumer can't tell which records belonged to the same atomic batch. If a downstream system needed to replay batches as transactions (e.g., applying all-or-nothing to a B-tree), it would need either:
- A batch-start marker (adding a new OP type)
- An explicit batch ID on each record
- Post-hoc grouping by scanning for COMMIT records and walking backward through sequence numbers

## Topics to Explore

- [function] `write-ahead-log/wal.py:iterate` — Compare iterate (raw record stream including COMMIT) with replay (filtered application-level view) to understand the two access patterns
- [function] `log-structured-merge-tree/lsm.py:_replay_wal` — How the LSM tree consumes WAL replay output to rebuild its memtable after crash recovery
- [general] `wal-atomicity-vs-sector-tearing` — Whether the single-buffer write in append_batch truly provides atomicity guarantees on all filesystems, or if partial sector writes can corrupt mid-batch
- [file] `write-ahead-log/tester_test_wal.py` — Additional test scenarios that may cover edge cases not in the main test file
- [general] `batch-start-marker-design` — How databases like PostgreSQL and SQLite use explicit transaction-begin markers to support true batch grouping during replay

## Beliefs

- `wal-replay-is-flat` — `replay()` returns all valid PUT/DELETE records without grouping by batch; COMMIT and CHECKPOINT records are filtered out, so the consumer has no visibility into which records belonged to the same atomic batch
- `wal-batch-atomicity-via-single-write` — `append_batch` achieves atomicity by serializing all batch records plus the COMMIT into a single buffer and issuing one `fd.write()` call followed by a forced fsync, rather than using a begin-transaction marker
- `wal-corruption-stops-replay` — Replay halts at the first CRC mismatch or truncated read; records after corruption are silently discarded, which means a crash mid-batch drops the entire incomplete batch
- `wal-no-nested-batches` — The WAL format has no batch-start marker and no nesting support; the lock held during `append_batch` prevents interleaving, making nested or concurrent batches structurally impossible

