# Topic: How production systems like LevelDB/RocksDB track "bottommost level" to decide when tombstone removal is safe during leveled compaction

**Date:** 2026-05-29
**Time:** 07:00

# Bottommost-Level Tracking for Safe Tombstone Removal

## What the Codebase Shows

The short answer: **this codebase does not implement bottommost-level tracking**. Both implementations use simplified tombstone removal strategies that sidestep the problem entirely. This is a meaningful gap worth understanding, because it highlights exactly why production systems need the concept.

### The Two Implementations

**`log-structured-merge-tree/lsm.py`** takes the simplest approach: its `compact()` method merges *all* SSTables into a single output file. Since there's nothing below the result, every tombstone is safe to drop. You can see this at lines 295–299 — after merging, it filters out tombstones unconditionally:

```python
# Build result, excluding tombstones        (line 295)
...
if v != TOMBSTONE:                           (line 299)
```

This works because "merge everything" is trivially the bottommost compaction — there are no older SSTables that might still hold the key below.

**`sstable-and-compaction/sstable.py`** has a leveled compaction strategy with `_max_levels = 7` (line 309) and iterates levels at lines 386 and 420. However, tombstone removal is controlled by an explicit `remove_tombstones` boolean passed to `merge_sstables` (visible in the test at `test_sstable.py:44`):

```python
merged = merge_sstables([reader2, reader], merged_path, remove_tombstones=True)
```

The caller decides. There's no logic that inspects whether the compaction output lands at the deepest occupied level.

## How Production Systems Actually Do It

In LevelDB and RocksDB, the rule is: **a tombstone can only be dropped if there is no older copy of that key in any level below the compaction output**. The mechanism has three parts:

1. **Level metadata tracking**: Each SSTable knows its level assignment. The `VersionSet` / `VersionStorageInfo` maintains which SSTables exist at each level and their key ranges.

2. **Bottommost detection at compaction time**: When the compaction picker selects input files, it checks whether any SSTable at a *lower* level overlaps the key range being compacted. If no lower level has overlapping files, this compaction is "bottommost" for those keys.

3. **Per-key decision during merge**: Even within a "bottommost" compaction, the decision is per-key. A tombstone for key `K` is only dropped if no SSTable at levels `L+1` through `L_max` contains `K` in its key range. RocksDB tracks this with `BottommostLevelCompaction` options and the `is_bottommost_level` flag on `CompactionJob`.

### Why This Matters

If you drop a tombstone too early — say, during a compaction at level 2 while an older PUT for the same key still sits at level 4 — the deleted key *reappears*. The tombstone was the only thing suppressing it. This is a data correctness bug, not a performance issue.

## What's Missing From These Observations

- **Lines 200–438 of `sstable.py`**: The full `CompactionManager` leveled compaction logic is cut off. The actual merge/compaction execution and tombstone handling during leveled compaction aren't visible.
- **No per-key overlap check**: Neither implementation checks whether lower levels contain overlapping key ranges before dropping tombstones.
- **No `VersionSet` equivalent**: There's no manifest or version tracking that maps SSTables to levels with key-range metadata, which is the foundation for the bottommost check in LevelDB/RocksDB.

---

## Topics to Explore

- [file] `sstable-and-compaction/sstable.py` — Read lines 200–438 to see the full `CompactionManager` leveled compaction logic, particularly how it picks files and handles merges across levels
- [function] `log-structured-merge-tree/lsm.py:compact` — The full compaction method shows how the "merge everything" strategy avoids the bottommost problem entirely
- [general] `leveldb-version-set` — Study LevelDB's `VersionSet` and `VersionEdit` to see how production systems track which SSTables exist at which levels with key ranges
- [general] `rocksdb-bottommost-compaction` — RocksDB's `BottommostLevelCompaction` enum and `CompactionJob::is_bottommost_level` flag show the production implementation of this concept
- [function] `sstable-and-compaction/sstable.py:merge_sstables` — Understand how the `remove_tombstones` flag works during multi-way merge and why a caller-controlled flag is insufficient for correctness in leveled compaction

## Beliefs

- `lsm-compact-removes-all-tombstones` — `LSMTree.compact()` unconditionally drops all tombstones because it merges all SSTables into one, making every compaction trivially "bottommost"
- `merge-sstables-tombstone-flag-is-caller-controlled` — `merge_sstables` accepts a `remove_tombstones` boolean from the caller rather than computing safety from level metadata, so correctness depends entirely on the caller passing the right value
- `sstable-compaction-manager-lacks-bottommost-detection` — The `CompactionManager` in `sstable.py` tracks `_max_levels` (default 7) but has no logic to check whether lower levels contain overlapping key ranges before dropping tombstones
- `premature-tombstone-removal-causes-data-resurrection` — Dropping a tombstone during compaction at level N while an older PUT for the same key exists at level N+k causes the deleted key to reappear in reads — a correctness violation, not merely a performance issue

