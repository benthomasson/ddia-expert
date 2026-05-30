# Topic: The WAL `commit` method syncs the data file then truncates the WAL; explore whether the ordering guarantees are correct if a crash hits between sync and truncate

**Date:** 2026-05-29
**Time:** 10:59

# WAL Commit Ordering: Sync-Then-Truncate Safety Analysis

## The Key Finding: There Is No `commit` Method

The grep for `def commit|def recover|def replay` returned **zero matches** in the WAL module. The `WriteAheadLog` class in `write-ahead-log/wal.py` has no `commit` method — and no other module imports the WAL (the grep for `from.*wal import|import.*wal` found only test files). The "sync data file, then truncate WAL" pattern described in the question **does not exist in this codebase yet**. It would live in a higher-level storage engine that composes this WAL with a data file.

That said, the WAL does expose the building blocks — `truncate()` and `checkpoint()` — and analyzing them reveals a real concern.

## What `truncate` Actually Does

Starting at `write-ahead-log/wal.py:178`, the `truncate(up_to_seq)` method:

1. **Flushes and fsyncs** the current WAL file descriptor
2. **Closes** the fd
3. **Iterates** over every `.wal` file, keeping only records with `seq_num > up_to_seq`
4. Either deletes the file (if no records survived) or rewrites it with only the kept records

The code was cut off at line 200, but the visible portion shows the critical structure.

## The Ordering Guarantee Analysis

If a caller were to implement the "sync data, then truncate WAL" pattern, the safety question comes down to: **what happens on crash between those two steps?**

### Crash between data sync and WAL truncate → Safe (idempotent replay)

This is actually the **correct** failure mode. If the data file is synced but the WAL is not yet truncated, recovery replays WAL records that have *already* been applied. As long as WAL replay is **idempotent** (PUT overwrites, DELETE removes), this produces the correct state — you just do redundant work.

### The real danger would be the reverse: truncate WAL *before* data sync

If the WAL were truncated first, a crash before data sync would lose committed operations with no recovery path. The "sync first, truncate second" ordering is the textbook-correct approach.

### Missing Guarantee: `fsync` on Rewritten WAL Files

Inside `truncate()` at line ~185–200, when the method rewrites WAL files with only the kept records, the observation was cut off before we can see whether those **rewritten files are fsynced**. If they aren't, a crash during truncation could leave a WAL file with partial content — the old data overwritten but the new data not yet durable. This would silently lose WAL records.

### Missing Guarantee: Directory `fsync` After File Deletion

When `truncate` removes empty WAL files, the directory entry removal isn't durable until the **directory itself** is fsynced. Without `os.fsync(os.open(self._dir, os.O_RDONLY))`, a crash could "resurrect" a deleted WAL file, leading to stale replays. This is the same class of bug as the well-known ext4 rename-without-directory-fsync issue.

### The `checkpoint` Method Is Just a Marker

The `checkpoint()` method at line 170 writes an `OP_CHECKPOINT` record and forces a sync, but it's purely a marker. It doesn't trigger any data flush or WAL truncation on its own — a caller must coordinate those steps.

## Summary

The sync-then-truncate ordering described in the question **would be correct** if implemented, because the failure mode (crash between sync and truncate) is safe — it just replays already-applied operations. The real risks are in the `truncate()` implementation itself: the observations were truncated before we could verify whether rewritten WAL files are fsynced and whether directory entries are fsynced after deletion. Those are the crash-safety gaps worth auditing.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:truncate` — The observation was cut off at line 200; read the full method to verify whether rewritten WAL files are fsynced before the old versions are unlinked
- [general] `directory-fsync-gap` — Investigate whether any WAL file creation or deletion is followed by an fsync of the parent directory — a common durability gap on Linux filesystems
- [function] `write-ahead-log/wal.py:replay` — The `replay` method was not included in the observations; understanding its idempotency guarantees is essential for validating the sync-then-truncate pattern
- [general] `wal-integration-with-storage-engine` — No module currently imports the WAL for production use; explore how a storage engine (e.g., the B-tree or LSM modules) should compose `checkpoint`, `truncate`, and data file sync into a correct commit protocol
- [function] `write-ahead-log/wal.py:_rotate` — Lines 107–116 show rotation fsyncs the old file but doesn't fsync the directory after creating the new file, which could lose the new file on crash

## Beliefs

- `wal-no-commit-method` — The `WriteAheadLog` class has no `commit` method; sync-then-truncate coordination must be implemented by a higher-level storage engine that composes the WAL
- `truncate-fsyncs-before-rewrite` — `truncate()` fsyncs and closes the current WAL fd before iterating over files, ensuring all pending writes are durable before any records are removed
- `wal-not-imported-outside-tests` — No production module imports `WriteAheadLog`; it is only used in `test_wal.py` and `tester_test_wal.py`, meaning crash-safe commit coordination is unimplemented
- `rotate-missing-directory-fsync` — `_rotate()` at line 107 fsyncs the old WAL file but does not fsync the parent directory after creating the new file, leaving a durability gap where the new file could be lost on crash

