# Topic: Find the code that replays WAL records and verify it actually groups records by COMMIT boundaries rather than applying them individually

**Date:** 2026-05-29
**Time:** 12:12

# WAL Replay and COMMIT Boundary Grouping

## The Short Answer: Replay Does NOT Group by COMMIT Boundaries

The `replay()` method's implementation is in `write-ahead-log/wal.py` **lines 201–264**, which were not captured in the observations. However, the test suite provides strong evidence of its behavior.

## What We Can See

The WAL defines four record types (`wal.py:10-13`):

```python
OP_PUT = 1
OP_DELETE = 2
OP_COMMIT = 3
OP_CHECKPOINT = 4
```

The `append_batch()` method (`wal.py:151-166`) writes individual operations followed by a COMMIT record, all in a single atomic write under a lock. The COMMIT acts as a durability marker — if a crash happens mid-batch, the incomplete batch will lack its COMMIT record on disk.

## Evidence from Tests That Replay Is a Flat Filter, Not a Transaction Grouper

The critical test is `test_basic` (`test_wal.py:7-28`):

```python
# 3 individual records (no COMMIT wrapping them)
seq1 = wal.append("PUT", "user:1", "alice")   # seq 1
seq2 = wal.append("PUT", "user:2", "bob")     # seq 2
seq3 = wal.append("DELETE", "user:1")          # seq 3

# 1 batch: 3 ops + 1 COMMIT = 4 records
seq_commit = wal.append_batch([...])           # seq 4,5,6 (ops) + 7 (COMMIT)

# checkpoint                                    # seq 8
cp_seq = wal.checkpoint()

records = wal.replay()
assert len(records) == 6  # <-- this is the key line
```

If replay grouped by COMMIT boundaries, it would return only the **3 batch operations** (the ones followed by a COMMIT). Instead it returns **6 records** — the 3 individually appended records *plus* the 3 batch operations. The COMMIT and CHECKPOINT records themselves are filtered out.

Compare with `iterate()` (`test_wal.py:86-92`), which returns **all** records including COMMIT:

```python
all_recs = list(wal.iterate())
assert len(all_recs) == 4          # 1 individual + 2 batch ops + 1 COMMIT
assert all_recs[3].op_type == "COMMIT"
```

## What This Means

`replay()` is a **flat filter**: it returns every PUT/DELETE record in sequence order, stripping out COMMIT and CHECKPOINT markers. It does **not**:

- Skip uncommitted records (individual appends without COMMIT are replayed)
- Group records into transaction batches
- Verify that a batch has a matching COMMIT before including its operations

This design means that if a crash occurs mid-`append_batch`, the partial records *would still be replayed* — the COMMIT record's absence doesn't cause the preceding operations to be excluded. The COMMIT marker is essentially metadata for downstream consumers to interpret, not an atomicity boundary enforced by replay.

> **The observations are incomplete.** Lines 201–264 of `wal.py` contain the actual `replay()` and `iterate()` implementations. The analysis above is inferred from test behavior and is high-confidence, but reading those lines would confirm the exact filtering logic.

## Topics to Explore

- [function] `write-ahead-log/wal.py:replay` — Read the actual implementation to confirm whether it truly ignores COMMIT boundaries or has any conditional logic not captured in tests
- [function] `write-ahead-log/wal.py:iterate` — Compare iterate vs replay to understand the raw-vs-filtered record access pattern
- [general] `crash-safety-gap` — Investigate whether the lack of COMMIT-boundary enforcement in replay is an intentional simplification or a correctness bug — a real WAL should skip uncommitted partial batches
- [function] `write-ahead-log/wal.py:append_batch` — Study how the atomic write buffer ensures all-or-nothing at the OS level, even though replay doesn't enforce it logically
- [file] `write-ahead-log/test_wal.py` — No test covers the partial-batch crash scenario (write ops without COMMIT, then replay) — this is the missing test that would expose the gap

## Beliefs

- `wal-replay-flat-filter` — `replay()` returns all PUT/DELETE records regardless of COMMIT boundaries; individually appended records without a COMMIT are included in replay results
- `wal-commit-is-metadata-only` — The COMMIT record serves as a durability/sync marker in `append_batch` but is not enforced as a transaction boundary during replay
- `wal-iterate-vs-replay` — `iterate()` returns all record types including COMMIT and CHECKPOINT; `replay()` filters those out and only returns data operations
- `wal-no-partial-batch-test` — The test suite lacks a test for crash-during-batch (partial write without COMMIT record), leaving the replay-of-uncommitted-ops behavior unverified

