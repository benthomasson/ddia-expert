# Topic: RocksDB's `BottommostLevelCompaction` enum values (`kSkip`, `kIfHaveCompactionFilter`, `kForce`, `kForceOptimized`) control when to trigger expensive bottom-level compaction for space reclamation

**Date:** 2026-05-29
**Time:** 13:48

# RocksDB's `BottommostLevelCompaction` — Not Present in This Codebase

**The observations are insufficient to explain this topic.** The grep for `BottommostLevelCompaction|kSkip|kIfHaveCompactionFilter|kForce|kForceOptimized` returned **zero matches**. This codebase contains Python reference implementations of LSM tree compaction, not RocksDB bindings or RocksDB-style configuration enums.

## What the Codebase *Does* Implement

The two compaction modules take a much simpler approach to the problem that `BottommostLevelCompaction` solves in RocksDB:

### 1. Full-merge compaction with unconditional tombstone removal

In `log-structured-merge-tree/lsm.py:319`, `compact()` merges **all** SSTables into one and **always** removes tombstones (line 340):

```python
# Remove tombstones during compaction
if v != TOMBSTONE:
    merged.append((k, v))
```

This is triggered automatically when the SSTable count exceeds `compaction_threshold` (line 316). There's no concept of levels — it's a flat merge of everything, which is closest to RocksDB's `kForce` behavior: always compact the bottom, always reclaim space.

### 2. Leveled compaction with configurable strategies

In `sstable-and-compaction/sstable.py`, the `CompactionManager` supports both `size_tiered` and `leveled` strategies. The test at `test_sstable.py:103-118` shows leveled compaction promoting L0 SSTables to L1. The `merge_sstables` function accepts a `remove_tombstones` flag (`test_sstable.py:44`), giving callers control over whether to reclaim space — a coarse analog to the skip/force distinction.

### What's Missing

To explain RocksDB's `BottommostLevelCompaction` enum properly, you'd need:

- **A multi-level LSM with a distinguished bottommost level** — where tombstones can only be safely dropped because no older data exists below. The current implementations either have no levels (`lsm.py`) or basic L0→L1 promotion (`sstable.py`).
- **Compaction filters** — user-defined callbacks that transform or drop entries during compaction. This is what `kIfHaveCompactionFilter` gates on. Neither module has this concept.
- **The cost/benefit distinction between `kForce` and `kForceOptimized`** — in RocksDB, `kForceOptimized` skips bottommost files that contain no tombstones or overwrites, avoiding rewriting data that won't shrink. The reference implementations have no such optimization; they rewrite everything unconditionally.

In short: the codebase demonstrates *why* compaction needs tombstone removal (see `lsm.py:265-266` for delete-via-tombstone, `lsm.py:340` for removal during compaction, and `test_lsm.py:69-87` proving correctness), but it doesn't model the **when-to-trigger** policy decisions that `BottommostLevelCompaction` encodes.

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:compact` — The full-merge compaction implementation; compare its simplicity with RocksDB's multi-level approach
- [function] `sstable-and-compaction/sstable.py:merge_sstables` — K-way merge with optional tombstone removal; the `remove_tombstones` flag is the closest analog to skip/force
- [file] `sstable-and-compaction/sstable.py` — `CompactionManager` with size-tiered and leveled strategies; read it to understand what a level-aware compaction policy looks like
- [general] `rocksdb-compaction-filters` — RocksDB's `CompactionFilter` API is the mechanism that `kIfHaveCompactionFilter` checks for; understanding it explains why this enum value exists
- [general] `tombstone-lifecycle-in-leveled-lsm` — Why tombstones can only be safely dropped at the bottommost level, and how this constraint motivates the entire enum

## Beliefs

- `lsm-compact-always-removes-tombstones` — `LSMTree.compact()` unconditionally strips tombstones during merge, with no policy control over whether to skip space reclamation (lsm.py:340)
- `no-rocksdb-bottommost-level-enum` — The codebase contains no `BottommostLevelCompaction` enum or equivalent; compaction policy is either threshold-based (lsm.py:316) or strategy-based (sstable.py's `CompactionManager`)
- `merge-sstables-tombstone-flag` — `merge_sstables` accepts a `remove_tombstones` boolean, providing binary skip/force control but no intermediate policies like filter-conditional or optimized-force
- `lsm-compaction-is-flat-merge` — `LSMTree.compact()` merges all SSTables into a single output regardless of level, unlike RocksDB which distinguishes bottommost-level compaction from inter-level compaction

