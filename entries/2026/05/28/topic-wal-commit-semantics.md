# Topic: The replay method ignores COMMIT markers — investigate whether this is intentional simplification or a bug, and what the implications are for batch atomicity guarantees

**Date:** 2026-05-28
**Time:** 18:17

I can work from the observations provided. The replay method starts at line 212 and the key comments are visible in the grep results. Here's the analysis:

---

# Replay Method and COMMIT Markers: Intentional Simplification with Real Implications

## What the code does

The WAL has two methods for reading records back:

- **`iterate()`** — yields *all* raw records, including COMMIT and CHECKPOINT markers. The test at `test_wal.py:91-92` confirms this: a standalone PUT + a 2-op batch produces 4 records (3 data ops + 1 COMMIT).

- **`replay()`** (`wal.py:212`) — returns only PUT/DELETE records, filtering out COMMIT and CHECKPOINT. The comments at lines 220–222 are explicit:

  ```
  # Individual writes (no COMMIT needed) and batch writes (have COMMIT)
  # marker. All PUT/DELETE records are included; COMMIT/CHECKPOINT are
  ```

  The test at `test_wal.py:22` confirms: 3 individual ops + 3 batch ops + 1 COMMIT + 1 CHECKPOINT = 8 total records, but `replay()` returns 6 (only the data ops).

## Is this intentional?

**Yes — it's a deliberate simplification, documented in the comments.** But it sacrifices a real guarantee.

## The atomicity gap

The issue becomes clear when you trace what happens during a crash mid-batch:

1. **`append_batch()`** (`wal.py:154–163`) builds all records (including the trailing COMMIT) into a single `bytearray`, then calls `self._fd.write(bytes(buf))` once. This provides *some* OS-level atomicity — small batches likely land in a single disk write.

2. **But for large batches**, the OS may flush partial data to disk before crashing. If a crash interrupts the write after some batch ops but before the COMMIT record, the WAL on disk contains orphaned data records with no COMMIT.

3. **On recovery, `replay()` ignores COMMIT markers entirely.** It returns all PUT/DELETE records that pass CRC validation, whether or not their batch was committed. Those orphaned partial-batch records would be replayed as if they were valid, **violating batch atomicity**.

A correct crash-recovery replay would need to:
- Track which records belong to which batch (by sequence number ranges)
- Only replay batch records if a corresponding COMMIT record exists
- Replay individual (non-batch) records unconditionally

## Why this is acceptable here

This is a reference implementation for DDIA concepts. The code demonstrates the *structure* of WAL batching — how a COMMIT record seals a group of operations, how `append_batch` uses a single write call for durability, how `iterate()` preserves the full record stream. The simplified `replay()` prioritizes a clean API for the common case (give me all the data operations) over production-grade crash recovery.

The fact that `iterate()` exists and *does* preserve COMMIT markers means a caller who needs real atomicity guarantees can build correct replay logic on top of it.

## Summary

| Aspect | Status |
|---|---|
| Documented? | Yes (lines 220–222) |
| Bug in normal operation? | No — `append_batch` writes atomically via single `write()` |
| Bug in crash recovery? | Yes — partial batches would replay without atomicity |
| Severity for a reference impl? | Low — demonstrates the concept; production code would check COMMITs |

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:iterate` — The raw record iterator that *does* preserve COMMIT markers; understanding the contrast with `replay()` clarifies the design intent
- [function] `write-ahead-log/wal.py:append_batch` — How the single-write atomicity trick works and where it breaks down for large batches
- [function] `write-ahead-log/wal.py:truncate` — Whether truncation respects batch boundaries or can split a committed batch
- [file] `log-structured-merge-tree/lsm.py` — Contains the other `replay` method (line 28); worth checking if the LSM tree consumes WAL replay and whether it would be affected by partial batches
- [general] `wal-crash-recovery-correctness` — Building a correct replay that groups records by batch and checks for COMMIT before applying, using `iterate()` as the foundation

## Beliefs

- `wal-replay-filters-commit-checkpoint` — `replay()` returns only PUT/DELETE records; COMMIT and CHECKPOINT markers are intentionally excluded and never reach the caller
- `wal-iterate-preserves-all-ops` — `iterate()` yields every record including COMMIT and CHECKPOINT, providing the raw stream needed for commit-aware recovery logic
- `wal-batch-single-write` — `append_batch()` serializes all ops plus the COMMIT marker into one `bytearray` and issues a single `fd.write()`, relying on OS write atomicity for small batches
- `wal-replay-no-atomicity-check` — `replay()` does not verify that batch operations have a corresponding COMMIT record, so partial batches from a crash would be replayed as if committed

