# Topic: Read RocksDB's `CompactionJob::SetupBottomMostLevel()` to see how it determines whether tombstones can be dropped based on overlapping key ranges in lower levels

**Date:** 2026-05-29
**Time:** 13:48

Now I have enough context to give a thorough explanation. Here's the full picture.

---

# RocksDB's `Compaction::SetupBottomMostLevel()` — Tombstone Drop Safety via Key Range Overlap

## The Problem It Solves

Before dropping a tombstone (delete marker) during compaction, RocksDB must answer one question: **does any level below the compaction output contain a file whose key range overlaps with this tombstone's key?** If yes, the tombstone must be preserved — dropping it would cause the older PUT in the lower level to "reappear," a correctness bug called **data resurrection**.

As documented in this knowledge base (`entries/2026/05/29/topic-leveled-compaction-tombstone-policy.md:44`):

> Dropping a tombstone during compaction at level N while an older PUT for the same key exists at level N+k causes the deleted key to reappear in reads — a correctness violation, not merely a performance issue.

## Where It Lives in RocksDB

The function lives on the **`Compaction` class** (not `CompactionJob` — a common confusion), in `db/compaction/compaction.cc`. `CompactionJob` *consumes* the result via `compaction->bottommost_level()`, but the decision logic belongs to `Compaction::SetupBottomMostLevel()`.

### The Algorithm

`SetupBottomMostLevel()` runs during compaction setup, after the compaction picker has selected input files but before any keys are merged. It works in three steps:

**Step 1 — Trivial case: output is the last level.**
If `output_level_ == number_levels_ - 1`, there are physically no levels below. Set `bottommost_level_ = true` and return. This is the fast path for the deepest level.

**Step 2 — Compute the compaction's key range.**
Collect the smallest and largest user keys across all input files (from both the input level and the output level, since leveled compaction pulls overlapping files from the target level too). These form the `[smallest_key, largest_key]` interval that the compaction covers.

**Step 3 — Scan levels below the output for overlap.**
For each level from `output_level_ + 1` down to `number_levels_ - 1`:
- Query the `VersionStorageInfo` (the immutable snapshot of the LSM tree's file layout) for any SSTable at that level whose key range intersects `[smallest_key, largest_key]`.
- RocksDB uses `VersionStorageInfo::OverlapInLevel()` or `RangeMightExistAfterSortedRun()` to do this efficiently — since levels 1+ maintain non-overlapping, sorted file ranges, this is a binary search, not a linear scan.
- If **any** overlap is found at any lower level, set `bottommost_level_ = false` and return immediately.
- If no level has overlapping files, set `bottommost_level_ = true`.

### How CompactionJob Uses the Result

In `CompactionJob::ProcessKeyValueCompaction()` (the inner merge loop in `compaction_job.cc`), the `bottommost_level_` flag gates several behaviors:

1. **Tombstone dropping**: If `bottommost_level_ == true`, delete markers and single-delete markers are eligible for removal during the merge. If `false`, they must be written to the output.
2. **Merge operand collapsing**: For merge operators, the bottommost flag tells the engine it can produce a final `Put` value rather than keeping partial merge operands.
3. **Sequence number zeroing**: At the bottommost level, keys' internal sequence numbers can be set to zero (since there's nothing below to compare against), which improves compression.

### The `BottommostLevelCompaction` Enum

Orthogonal to `SetupBottomMostLevel()` is the `BottommostLevelCompaction` enum (in `include/rocksdb/advanced_options.h`), which controls **when the engine triggers** compaction of the bottommost level:

| Value | Behavior |
|-------|----------|
| `kSkip` | Never compact the bottommost level unless forced — saves CPU but delays space reclamation |
| `kIfHaveCompactionFilter` | Compact bottommost only if a `CompactionFilter` is installed (for TTL expiration, etc.) |
| `kForce` | Always compact the bottommost level when eligible |
| `kForceOptimized` | Like `kForce`, but only for files that actually contain tombstones or merge operands |

This enum answers "**should** we compact the bottom level?" while `SetupBottomMostLevel()` answers "**is** this compaction at the bottom level?" — related but distinct questions.

## How This Contrasts with the DDIA Reference Implementations

The knowledge base documents two reference implementations that handle this differently:

**`log-structured-merge-tree/lsm.py`** — `compact()` merges *all* SSTables into one file (`entries/2026/05/29/topic-rocksdb-bottommost-compaction.md:16`). This is trivially bottommost — there's nothing below — so tombstone removal is always safe. No overlap check needed.

**`sstable-and-compaction/sstable.py`** — `merge_sstables()` takes an explicit `remove_tombstones` boolean (`entries/2026/05/28/sstable-and-compaction-sstable-merge_sstables.md:37`). The caller decides, and as noted at line 100: "neither call site passes `remove_tombstones=True`." The `CompactionManager` has `_max_levels = 7` but no `VersionStorageInfo` equivalent — no per-level file metadata to query for overlaps. The gap is documented as belief `sstable-compaction-manager-lacks-bottommost-detection` in `topic-leveled-compaction-tombstone-policy.md:66`:

> The `CompactionManager` tracks `_max_levels` (default 7) but has no logic to check whether lower levels contain overlapping key ranges before dropping tombstones.

## The Critical Invariant

The entire system rests on one invariant: **`VersionStorageInfo` must be an accurate, atomic snapshot of which files exist at which levels with what key ranges.** If this metadata drifts from reality — a file is added but not registered, or a registered file is deleted — `SetupBottomMostLevel()` could produce a wrong answer, leading to either:
- **False positive** (`bottommost = true` when it isn't): tombstones dropped unsafely, data resurrection.
- **False negative** (`bottommost = false` when it is): tombstones preserved unnecessarily, wasting space but not breaking correctness.

RocksDB ensures consistency through its `VersionSet` mechanism — immutable, reference-counted `Version` objects that are atomically installed via `VersionEdit` records written to the `MANIFEST` WAL. This is the same pattern documented in `entries/2026/05/29/topic-leveldb-manifest-pattern.md`.

---

## Topics to Explore

- [function] `sstable-and-compaction/sstable.py:CompactionManager._lcs_compact` — The leveled compaction implementation that would need bottommost detection if tombstone removal were enabled — trace how it selects overlapping files across levels
- [general] `rocksdb-version-storage-info` — `VersionStorageInfo` is the data structure that `SetupBottomMostLevel()` queries for per-level file ranges; understanding it is essential to understanding the overlap check
- [general] `rocksdb-compaction-iterator` — `CompactionIterator` in `db/compaction/compaction_iterator.cc` is where the per-key tombstone/merge-operand decisions are actually made using the `bottommost_level_` flag
- [function] `log-structured-merge-tree/lsm.py:compact` — The "merge everything" strategy that makes bottommost detection unnecessary — a useful contrast for understanding why leveled compaction needs it
- [general] `snapshot-aware-tombstone-retention` — Even at the bottommost level, RocksDB retains tombstones visible to open snapshots; `SetupBottomMostLevel()` is a necessary but not sufficient condition for dropping a tombstone

## Beliefs

- `setup-bottommost-scans-below-output-level` — `Compaction::SetupBottomMostLevel()` iterates levels from `output_level_ + 1` to `number_levels_ - 1`, checking each for files overlapping the compaction's key range via `VersionStorageInfo`; any overlap means `bottommost_level_ = false`
- `bottommost-false-negative-is-safe` — A false negative from `SetupBottomMostLevel()` (reporting non-bottommost when it is) wastes space by preserving unnecessary tombstones but never causes data corruption; a false positive causes data resurrection
- `bottommost-enables-three-optimizations` — The `bottommost_level_` flag gates tombstone dropping, merge operand collapsing, and sequence number zeroing during `ProcessKeyValueCompaction` — all three are unsafe if the flag is wrong
- `bottommost-level-enum-is-trigger-not-detection` — `BottommostLevelCompaction` (kSkip/kForce/etc.) controls whether to *schedule* bottommost compaction, while `SetupBottomMostLevel()` determines whether a given compaction *is* bottommost — they answer orthogonal questions

