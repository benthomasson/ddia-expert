# Topic: The two compaction strategies tested here have very different tradeoffs in write amplification, space amplification, and read performance (DDIA Chapter 3)

**Date:** 2026-05-29
**Time:** 06:58

# Compaction Strategies: Size-Tiered vs. Leveled

This codebase implements two distinct compaction strategies in `sstable-and-compaction/sstable.py`, configured through the `CompactionManager` class, plus a simpler all-at-once compaction in `log-structured-merge-tree/lsm.py`. The two named strategies — **size-tiered** and **leveled** — map directly to the tradeoffs Kleppmann describes in DDIA Chapter 3.

## The Two Strategies in Code

### Size-Tiered Compaction (STCS)

Tested at `test_sstable.py:47`:

```python
manager = CompactionManager(tmpdir, strategy='size_tiered', min_threshold=2)
```

Size-tiered compaction groups SSTables by similar size and merges them when enough accumulate. The `min_threshold=2` parameter controls how many similarly-sized SSTables trigger a merge. The result is a new, larger SSTable — which will eventually be grouped with *other* large SSTables and merged again, creating progressively larger tiers.

### Leveled Compaction (LCS)

Tested at `test_sstable.py:108`:

```python
mgr = CompactionManager(lcs_dir, strategy='leveled', l0_compaction_trigger=2)
```

Leveled compaction organizes SSTables into numbered levels with non-overlapping key ranges (except L0). The `l0_compaction_trigger=2` means: once L0 has 2 SSTables, compact them into L1. The test verifies this promotion explicitly (`test_sstable.py:119`):

```python
assert result[0].level == 1
```

Each `SSTableMetadata` carries a `level` field (`sstable.py:38`), and the compaction manager uses it to track which level each SSTable belongs to.

## The Tradeoff Triangle

### Write Amplification

**Size-tiered wins.** Each SSTable is written once when flushed, then rewritten each time it participates in a merge. Since STCS only merges similarly-sized groups, a given entry gets rewritten ~O(log N) times across tiers. Leveled compaction is more aggressive — when an L0 SSTable overlaps with existing L1 data (which it almost always does, since L0 ranges overlap freely), the overlapping L1 SSTables must be read, merged, and rewritten. An entry can be rewritten O(L × fan-out) times as it propagates down through levels.

The LSM tree in `lsm.py` takes the extreme approach at line 340: it merges *all* SSTables into one during `compact()`, meaning every surviving entry gets rewritten every compaction cycle. This is the simplest strategy but has the worst write amplification at scale.

### Space Amplification

**Leveled wins.** STCS allows multiple SSTables to contain different versions of the same key simultaneously — the test at `test_sstable.py:34-43` demonstrates this: `path1` has `apple=red` and `path2` has `apple=green`. Both coexist on disk until compaction merges them. At worst, STCS can temporarily double space usage during compaction (the old and new SSTables overlap). Leveled compaction keeps key ranges non-overlapping within each level, so each key exists at most once per level, bounding space overhead.

### Read Performance

**Leveled wins.** With size-tiered compaction, a point lookup may need to check multiple SSTables at the same tier because their key ranges overlap freely. The test at `test_sstable.py:47-54` shows this: both `r1` and `r2` contain the key `apple`, so a lookup must check both (preferring the newer one via timestamp — `sstable.py:28` stores timestamps per entry).

With leveled compaction, each level beyond L0 has non-overlapping key ranges, so at most one SSTable per level needs to be checked. Combined with the sparse index (`sstable.py:56-57`, every `block_size` entries gets an index entry), this bounds the I/O per read to O(levels × 1 binary search + scan).

The `merge_sstables` function (tested at `test_sstable.py:90-98`) resolves conflicts by timestamp — `shared[0].value == 'val_4'` because timestamp `4.0` is highest — which is how both strategies determine which version of a key survives compaction.

## Summary Table

| Dimension | Size-Tiered | Leveled | `lsm.py` (full merge) |
|-----------|------------|---------|----------------------|
| Write amplification | Low | High | Very high at scale |
| Space amplification | High (overlapping ranges) | Low (non-overlapping per level) | Low after compaction, spikes during |
| Read performance | Slower (check many SSTables) | Faster (1 SSTable/level) | Best immediately after compaction |
| Best for | Write-heavy workloads | Read-heavy / space-constrained | Small datasets, learning |

## What's Missing from the Observations

The observations only include the first 200 lines of `sstable.py` (438 total), so the `CompactionManager` class itself — including the actual size-tiered grouping logic, the leveled promotion/overlap-detection code, and the merge orchestration — is not visible. The analysis above is based on the test behavior and DDIA's descriptions of these strategies. To fully verify the implementation's fidelity to the textbook algorithms, you'd want to read `sstable.py:200-438`.

## Topics to Explore

- [function] `sstable-and-compaction/sstable.py:CompactionManager` — The actual strategy implementations: how size-tiered groups SSTables by size, how leveled detects key-range overlaps between levels, and whether the fan-out ratio between levels is configurable
- [function] `log-structured-merge-tree/lsm.py:LSMTree.compact` — The full-merge compaction at line 340; compare its simplicity (and write amplification cost) against the two-strategy approach in `sstable.py`
- [function] `sstable-and-compaction/sstable.py:merge_sstables` — The multi-way merge that both strategies depend on; uses `heapq` for efficient k-way merging with timestamp-based conflict resolution
- [general] `bloom-filters-for-read-optimization` — Neither implementation includes Bloom filters, which DDIA describes as critical for reducing the number of SSTables checked during point lookups, especially under size-tiered compaction
- [file] `log-structured-merge-tree/lsm.py` — The WAL + memtable + flush pipeline (lines 1-200) that feeds entries into SSTables before compaction ever runs; understanding this write path is prerequisite to understanding why compaction strategy matters

## Beliefs

- `stcs-overlapping-key-ranges` — Size-tiered compaction allows multiple SSTables at the same tier to contain overlapping key ranges, requiring reads to check all of them
- `leveled-promotes-to-numbered-levels` — Leveled compaction promotes SSTables from L0 into higher numbered levels with non-overlapping key ranges within each level (verified by `result[0].level == 1` in the test)
- `timestamp-resolves-merge-conflicts` — Both the SSTable merge function and the LSM tree use entry timestamps to determine which version of a duplicate key survives compaction (newest wins)
- `lsm-compact-is-full-merge` — The `LSMTree.compact()` method merges all SSTables into a single new SSTable, unlike the incremental strategies in `CompactionManager`
- `sparse-index-bounds-scan-cost` — SSTable point lookups use a sparse index (one entry per `block_size` keys) with binary search to find the right block, then linear scan within the block

