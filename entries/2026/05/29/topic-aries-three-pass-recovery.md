# Topic: ARIES's analysis/redo/undo phases and why the undo phase disappears under NO-STEAL

**Date:** 2026-05-29
**Time:** 12:13

# ARIES Recovery Phases and the NO-STEAL Simplification

## What ARIES Does (and What This Codebase Doesn't)

ARIES (Algorithms for Recovery and Isolation Exploiting Semantics) is a three-phase crash recovery protocol: **analysis**, **redo**, **undo**. This codebase implements only the first two — and that's not accidental. The buffer management policy baked into these modules makes the third phase unnecessary.

## The Three Phases, Mapped to This Code

### 1. Analysis Phase — "What happened before the crash?"

The analysis phase scans the log from the last checkpoint forward to reconstruct two things: which transactions were active (uncommitted) at crash time, and which pages might be dirty on disk.

In `write-ahead-log/wal.py:85-96`, `_recover_seq_num()` is a stripped-down analysis phase:

```python
def _recover_seq_num(self) -> int:
    """Scan all WAL files to find the highest sequence number."""
    max_seq = 0
    for path in self._wal_files():
        with open(path, "rb") as f:
            while True:
                try:
                    rec = _read_record(f)
                    if rec is None:
                        break
                    max_seq = max(max_seq, rec.seq_num)
                except ValueError:
                    break
    return max_seq
```

It scans every WAL file to find the high-water mark. A full ARIES analysis phase would also build a **dirty page table** (which pages have unflushed changes) and a **transaction table** (which transactions were in-flight). This implementation skips both — it only needs the sequence number because its recovery model is simpler.

The `checkpoint()` method at line 169 writes a checkpoint record that `replay()` uses as a starting fence (see `test_wal.py:20-24`, where `replay(after_seq=cp_seq)` returns zero records). In ARIES, the checkpoint would also persist the dirty page table and transaction table to avoid scanning from the beginning of time.

### 2. Redo Phase — "Replay history to reach the pre-crash state"

The redo phase replays committed operations from the log to bring the database back to its state at the moment of the crash. This is a **repeating history** paradigm — ARIES redoes everything, including operations from uncommitted transactions, to reconstruct the exact pre-crash state.

The `replay()` method (referenced throughout `test_wal.py`) serves this role. In `test_crash_recovery` at `test_wal.py:29-41`:

```python
def test_crash_recovery():
    wal = WriteAheadLog(tmpdir, sync_mode="sync")
    wal.append("PUT", "k1", "v1")
    wal.append("PUT", "k2", "v2")
    # simulate crash — no close()
    wal2 = WriteAheadLog(tmpdir, sync_mode="sync")
    records = wal2.replay()
    assert len(records) == 2
```

The WAL is reopened without a clean shutdown, and `replay()` recovers both records. Note that `append_batch()` at `wal.py:154-166` writes all operations plus a COMMIT record atomically with a forced fsync — the COMMIT record acts as the durability boundary. Records without a matching COMMIT can be identified and skipped during replay.

### 3. Undo Phase — "Roll back uncommitted transactions"

This is where it gets interesting. **There is no undo phase in this codebase.** Grep for "undo" finds nothing. The MVCC module at `snapshot-isolation/mvcc_database.py` has no undo logic either — `abort()` just marks the transaction status and versions created by aborted transactions are filtered out by `_is_visible()` at line 76.

## Why Undo Disappears Under NO-STEAL

The undo phase exists in ARIES because of the **STEAL** buffer policy. STEAL means the buffer manager is allowed to flush a dirty page to disk **before** the transaction that modified it has committed. If the system crashes after flushing a dirty page but before the transaction commits, that page now contains uncommitted data on disk. The undo phase walks the log backward to reverse these changes.

Under **NO-STEAL**, dirty pages from uncommitted transactions are **never written to stable storage**. If a crash happens, uncommitted changes simply don't exist on disk — there's nothing to undo.

This codebase uses NO-STEAL implicitly in two ways:

1. **The WAL module** (`wal.py`) only writes log records, not data pages. The consuming modules (like the LSM tree at `log-structured-merge-tree/lsm.py`) rebuild state entirely from the log during recovery. There is no separate "data page" that could be flushed with uncommitted data.

2. **The MVCC module** (`mvcc_database.py`) is entirely in-memory. Versions live in `self._versions` (a Python dict at line 53). There are no data pages being flushed to disk at all. Uncommitted versions are simply never persisted — the `_is_visible()` check at line 76 filters them out based on transaction status, but even that filtering only applies to the in-memory state.

The tradeoff: NO-STEAL simplifies recovery (no undo needed) but limits how much data you can buffer — every dirty page from an active transaction must stay in memory until commit. STEAL is what allows large transactions in systems with limited RAM, at the cost of needing the undo phase.

## The FORCE vs NO-FORCE Axis

There's a companion dimension worth noting. The `append_batch()` method at `wal.py:165` calls `self._do_sync(force=True)` on commit — it forces the log to stable storage. But the question is whether **data pages** are forced to disk at commit time (FORCE) or not (NO-FORCE). ARIES was designed for NO-FORCE/STEAL — the hardest combination, needing both redo and undo. This codebase effectively operates at FORCE/NO-STEAL (or more precisely, log-only with no separate data pages), which needs only redo.

## Topics to Explore

- [file] `log-structured-merge-tree/lsm.py` — Shows how the WAL is consumed during crash recovery in a concrete storage engine; the LSM tree rebuilds its memtable from the WAL on startup, which is pure redo with no undo
- [function] `snapshot-isolation/mvcc_database.py:_is_visible` — The visibility check that makes undo unnecessary in MVCC; aborted transactions' versions are simply never visible, so there's nothing to roll back
- [function] `write-ahead-log/wal.py:append_batch` — The atomic commit boundary; understanding why the COMMIT record plus forced fsync guarantees all-or-nothing without undo
- [general] `steal-vs-no-steal-buffer-policies` — How real databases (PostgreSQL, InnoDB) use STEAL to handle transactions larger than memory, and the undo machinery that entails (undo tablespaces, rollback segments)
- [general] `aries-clr-compensation-log-records` — In full ARIES, the undo phase writes CLRs to avoid re-undoing on a crash during recovery; this is the mechanism that makes ARIES idempotent across repeated crashes

## Beliefs

- `wal-no-undo-recovery` — The WAL module implements analysis (sequence number recovery) and redo (replay) but has no undo phase; uncommitted data is never persisted to stable storage
- `wal-checkpoint-as-replay-fence` — `checkpoint()` writes a marker record that `replay(after_seq=)` uses to skip already-applied records, serving as a simplified ARIES checkpoint without dirty page or transaction tables
- `mvcc-abort-is-logical-not-physical` — `MVCCDatabase.abort()` marks transaction status but does not reverse any writes; `_is_visible()` filters out aborted versions at read time, eliminating the need for physical undo
- `wal-batch-commit-atomicity` — `append_batch()` writes all operations plus a COMMIT record in a single buffer with forced fsync, making the COMMIT record the durability boundary for transaction atomicity

