# Topic: How the checkpoint sequence number flows between the main data store and the WAL to ensure truncate is safe

**Date:** 2026-05-29
**Time:** 06:24

# Checkpoint Sequence Number Flow: WAL ↔ Data Store

## What the WAL Provides

The WAL in `write-ahead-log/wal.py` exposes two key methods that form a coordination protocol:

1. **`checkpoint()` (line 170)** — Increments the internal `_seq_num`, writes an `OP_CHECKPOINT` record to the log, force-syncs to disk, and **returns the checkpoint's sequence number**. This return value is the handoff point — it tells the caller "everything up to this number is now marked as checkpointed."

2. **`truncate(up_to_seq)` (line ~180)** — Accepts a sequence number and physically removes all records with `seq_num <= up_to_seq`. It flushes and fsyncs the current file before scanning, then rewrites each `.wal` file keeping only records above the cutoff. Files that become empty are deleted entirely.

The intended protocol is:

```
Data Store                          WAL
    |                                |
    |-- flush state to disk -------->|
    |-- wal.checkpoint() ----------->|  writes OP_CHECKPOINT, returns seq=N
    |<---- seq N --------------------|
    |                                |
    |  (store records N as its       |
    |   last-durable checkpoint)     |
    |                                |
    |-- wal.truncate(N) ------------>|  deletes all records where seq ≤ N
    |                                |
```

The sequence number is the **contract** between the two: the data store promises "I have durably applied everything up to N," and the WAL promises "I will only discard records ≤ N."

## How the Sequence Number Makes Truncation Safe

The monotonic counter (`_seq_num`, initialized at line 73 and recovered via `_recover_seq_num()` at line 84) guarantees a total ordering. The safety argument is:

- Every mutation gets a unique, increasing seq_num via `append()` (line 139) or `append_batch()` (line 148).
- `checkpoint()` gets a seq_num **after** all preceding mutations — so if checkpoint returns N, mutations 1..N-1 are guaranteed to precede it in the log.
- `truncate(N)` only removes records ≤ N. Any mutations appended after the checkpoint have seq_num > N and are preserved.
- On crash recovery, `_recover_seq_num()` (line 84) scans all WAL files to find the highest surviving seq_num, so the counter never goes backward.

The test at `write-ahead-log/test_wal.py:63-70` demonstrates this directly:

```python
wal.append("PUT", "a", "1")   # seq 1
wal.append("PUT", "b", "2")   # seq 2
wal.append("PUT", "c", "3")   # seq 3
wal.truncate(2)                # discard seq ≤ 2
records = wal.replay()
assert len(records) == 1       # only seq 3 survives
assert records[0].key == "c"
```

## What's Missing from the Observations

**The observations are insufficient to show the full flow.** Specifically:

- **No data store class was found** — the grep for `class.*Store|class.*Database|class.*Engine` returned zero matches. We can't see who calls `checkpoint()` and feeds its return value to `truncate()`.
- **The LSM tree call at `log-structured-merge-tree/lsm.py:314`** uses `self._wal.truncate()` with **no argument**, which doesn't match the WAL's `truncate(up_to_seq)` signature. This suggests the LSM either wraps a different WAL implementation or tracks the checkpoint seq internally and passes it differently.
- **No code shows the "store durably, then checkpoint, then truncate" three-step sequence** in a single caller. The WAL provides the primitives, but the coordination logic that ties them together isn't visible in these observations.

To fully answer the question, you'd need to read the LSM tree's flush/compaction code and whatever storage engine consumes this WAL module.

---

## Topics to Explore

- [file] `log-structured-merge-tree/lsm.py` — Contains `self._wal.truncate()` at line 314; likely shows the flush-then-truncate coordination the WAL module alone doesn't reveal
- [function] `write-ahead-log/wal.py:replay` — The recovery-side counterpart: how replayed records interact with checkpoint markers to rebuild state after a crash
- [function] `write-ahead-log/wal.py:_recover_seq_num` — Scans all WAL files on startup to restore the monotonic counter; understanding this is key to seeing why seq_nums never collide after crash
- [file] `unbundled-database/test_unbundled_database.py` — Uses `wal.truncate_before(3)` (a different API shape), suggesting an alternative WAL consumer with its own checkpoint coordination pattern
- [general] `checkpoint-vs-commit-semantics` — The WAL has both `OP_COMMIT` (for batch atomicity) and `OP_CHECKPOINT` (for truncation safety); understanding when each is used clarifies the two distinct durability guarantees

## Beliefs

- `wal-checkpoint-returns-seq` — `WriteAheadLog.checkpoint()` returns the sequence number it wrote, providing the caller the exact truncation boundary
- `wal-truncate-requires-explicit-seq` — `truncate(up_to_seq)` takes an explicit sequence number parameter; the WAL never decides what to truncate on its own
- `wal-seq-num-recovered-on-init` — On construction, `_recover_seq_num()` scans all WAL files to find the max seq_num, guaranteeing monotonicity across crashes
- `wal-truncate-fsyncs-before-scan` — `truncate()` flushes and fsyncs the current WAL file before scanning for records to remove, preventing data loss from buffered writes
- `lsm-wal-truncate-no-arg` — The LSM tree at `lsm.py:314` calls `self._wal.truncate()` with no sequence argument, indicating it uses a different WAL interface than `write-ahead-log/wal.py`

