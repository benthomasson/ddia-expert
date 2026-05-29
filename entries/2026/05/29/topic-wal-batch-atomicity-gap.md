# Topic: The gap between the docstring's claim ("skips uncommitted batches") and the implementation (returns all PUT/DELETE regardless of COMMIT presence)

**Date:** 2026-05-29
**Time:** 07:55

I now have a thorough understanding of the gap. Here's the explanation:

---

# The WAL Batch Atomicity Gap: Docstring vs. Implementation

## The Claim

The `replay()` method in `write-ahead-log/wal.py` (line ~212) carries a docstring that claims it "skips uncommitted batches." The companion method `iterate()` reinforces this framing in its own docstring, describing itself as providing "raw, unfiltered access" that includes "uncommitted batch operations," contrasting with `replay`, "which filters out non-data records and **skips uncommitted batches**."

This language promises that `replay()` is COMMIT-aware: it should identify which PUT/DELETE records belong to a batch, check whether a COMMIT record follows them, and exclude records from incomplete batches.

## What the Code Actually Does

The implementation does something much simpler. According to the inline comments at lines 220–222:

```
# Individual writes (no COMMIT needed) and batch writes (have COMMIT)
# marker. All PUT/DELETE records are included; COMMIT/CHECKPOINT are
```

`replay()` performs **two filters** and nothing more:

1. **Sequence number filter**: skip records where `seq_num <= after_seq`
2. **Op-type filter**: keep only `PUT` and `DELETE`; discard `COMMIT` and `CHECKPOINT`

There is no batch-tracking logic. No grouping of records by batch. No scan for a matching COMMIT. Every PUT and DELETE that passes CRC validation and is above the sequence threshold gets returned, period.

## Why This Matters

The gap has real consequences during crash recovery. Trace the failure scenario:

1. A caller invokes `append_batch()` with, say, a two-key money transfer: `[("PUT", "alice", "900"), ("PUT", "bob", "1100")]`.

2. `append_batch()` encodes all three records (2 PUTs + 1 COMMIT) into a single `bytearray` and issues one `write()` call. For small batches, the OS will likely write this atomically.

3. **But if the batch is large** — or if the OS flushes page-cache pages selectively — a crash can land the WAL in a state where the two PUT records are on disk but the COMMIT record is not.

4. On recovery, `replay()` reads those two orphaned PUTs, sees they pass CRC validation, and returns them. The caller applies `alice=900` and `bob=1100` as if the transaction committed. **Batch atomicity is violated.**

A correct implementation would need to:
- Identify batch boundaries (e.g., by sequence number ranges between COMMITs)
- Only emit batch records when the closing COMMIT is present
- Continue emitting individual (non-batch) writes unconditionally

## Why the Code Gets Away With It

This is a **reference implementation** for DDIA concepts, not production infrastructure. Three factors make the gap tolerable:

1. **Single-write atomicity**: `append_batch()` writes the entire buffer in one `fd.write()` call. For typical small batches, the OS will commit this atomically. The partial-write scenario requires either a large batch that spans multiple disk pages or a crash at a very unlucky moment.

2. **`iterate()` exists**: The raw iterator preserves COMMIT records. A caller who needs real atomicity can build commit-aware filtering on top of `iterate()` — the information isn't lost, it's just not used by `replay()`.

3. **The code is self-aware**: The inline comments explicitly describe what `replay()` does (returns all PUT/DELETE), and the exploration entries document this as a known simplification. The docstring is aspirational; the comments are accurate.

## The Docstring's Role

The docstring appears to describe the *intended* behavior of a fully-correct WAL replay, while the implementation takes a shortcut. This is a common pattern in reference implementations: the documentation describes the concept from DDIA, while the code demonstrates the structural elements (COMMIT records, single-write batching, fsync forcing) without implementing every production-grade invariant. The tests confirm this — `test_basic` in `test_wal.py` verifies that `replay()` returns the right count of PUT/DELETE records, but there is no test for crash-mid-batch recovery that would expose the atomicity gap.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:append_batch` — How the single-write trick provides probabilistic atomicity for small batches, and where it breaks down for large ones
- [function] `write-ahead-log/wal.py:_read_all_records` — The corruption-stopping iterator that `replay` delegates to; its early-termination behavior interacts with the atomicity gap (corruption mid-batch stops all subsequent records, but uncommitted batch records before the corruption point still get returned)
- [function] `write-ahead-log/wal.py:iterate` — The raw record stream that preserves COMMIT markers, which a production-grade replay could use to implement proper batch boundary tracking
- [file] `log-structured-merge-tree/lsm.py` — The LSM tree's `replay` method (line 28) consumes WAL replay during crash recovery; check whether it would be affected by partial batch replay
- [general] `commit-aware-replay-design` — Designing a correct replay that groups records by batch using sequence number ranges between COMMIT markers, handling edge cases like interleaved individual writes and nested batches

## Beliefs

- `replay-does-not-enforce-batch-atomicity` — Despite the docstring claiming replay "skips uncommitted batches," the implementation returns all PUT/DELETE records regardless of whether a matching COMMIT record exists; a crash mid-batch will replay partial batch records
- `wal-docstring-describes-intent-not-behavior` — The `replay()` docstring describes the conceptual DDIA behavior (committed-only replay), while the inline comments and implementation reflect the actual simplified behavior (all-data-ops replay)
- `iterate-enables-correct-replay` — `iterate()` preserves COMMIT and CHECKPOINT markers in the record stream, providing the raw material needed to build a commit-aware replay externally
- `batch-atomicity-is-fsync-level-only` — `append_batch`'s atomicity guarantee operates at the fsync level (all-or-nothing disk write via single `write()` call), not at the replay-filtering level; the two guarantees are independent and the second is missing

