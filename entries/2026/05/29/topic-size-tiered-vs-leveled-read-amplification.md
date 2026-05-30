# Topic: Compare how overlapping key ranges in size-tiered compaction amplify the cost of missing-key lookups versus leveled compaction's non-overlapping guarantees

**Date:** 2026-05-29
**Time:** 13:27

# Overlapping Key Ranges and Missing-Key Lookup Cost: Size-Tiered vs. Leveled Compaction

## The Core Problem

When you look up a key that **doesn't exist** in an LSM tree, the engine can't stop early — it has to prove the key is absent. The compaction strategy determines how many SSTables must be consulted to reach that proof, and this is where size-tiered and leveled compaction diverge dramatically.

## How the LSM Tree Searches

The `LSMTree.get()` method (`log-structured-merge-tree/lsm.py:247`) follows a fixed search order: check the memtable first, then walk SSTables from newest to oldest. For a **present** key, the search terminates at the first hit. For a **missing** key, every source must report "not found" before the engine can return `None`.

Each individual SSTable lookup (`lsm.py:117`) uses a sparse index with binary search to locate the right block, then scans linearly within that block. This isn't free — it involves file I/O and deserialization — but it's bounded per SSTable. The question is: **how many SSTables must you probe?**

## Size-Tiered Compaction: Overlapping Ranges Multiply the Cost

In size-tiered compaction (visible in `sstable-and-compaction/test_sstable.py:49–55` where a `CompactionManager` is created with `strategy='size_tiered'`), SSTables accumulate at the same tier and **their key ranges freely overlap**. Consider what happens with the test setup:

```python
# test_sstable.py:6-12 — SSTable 1 covers [apple, elderberry]
writer.add('apple', 'red', 1.0)
...
writer.add('elderberry', 'purple', 1.0)

# test_sstable.py:33-36 — SSTable 2 covers [apple, fig]  
writer2.add('apple', 'green', 2.0)
...
writer2.add('fig', 'purple', 2.0)
```

Both SSTables cover the range `[apple, elderberry]`. If you look up `"coconut"` (a missing key), you **cannot skip either SSTable** — both have key ranges that contain `"coconut"`. You must probe SSTable 2's sparse index, scan the relevant block, find nothing, then do the same in SSTable 1.

Now scale this up. The LSM tree's compaction logic (`lsm.py:315–317`) only triggers when the SSTable count crosses a threshold:

```python
if len(self._sstables) >= self._compaction_threshold:
    self.compact()
```

With a threshold of, say, 10 and writes that touch the same key space, you can have 9 SSTables all covering `[a, z]`. A missing-key lookup probes **all 9**. The cost is **O(N)** in the number of SSTables, and N grows between compaction cycles.

The `lsm.py:319` `compact()` method merges all SSTables into one, which temporarily fixes the problem — but the cycle repeats as new SSTables flush from the memtable.

## Leveled Compaction: Non-Overlapping Guarantees

Leveled compaction (tested at `test_sstable.py:104–119` with `strategy='leveled'`) enforces a structural invariant: **within each level (L1 and above), no two SSTables have overlapping key ranges.** Each SSTable "owns" a disjoint slice of the keyspace.

For a missing-key lookup, this changes the math fundamentally. At each level, you check which SSTable's `[min_key, max_key]` range could contain your key. Because ranges don't overlap, **at most one SSTable per level can be a candidate.** The `SSTableMetadata` fields (`sstable.py:37–39`) — `min_key`, `max_key` — enable this range check before any I/O:

```python
# sstable.py:37-39
min_key: str
max_key: str
entry_count: int
```

If the key falls outside every SSTable's range at a given level, you skip the entire level with no I/O at all. The worst case for a missing-key lookup becomes **O(L)** where L is the number of levels (typically 5–7), not the number of SSTables (which can be hundreds).

## The Amplification Factor

Here's a concrete comparison. Suppose you have 100 SSTables worth of data:

| | Size-Tiered | Leveled |
|---|---|---|
| **SSTables probed (missing key, worst case)** | Up to ~100 (all in one tier may overlap) | ~5–7 (one per level) |
| **I/O operations per probe** | Binary search on sparse index + block scan | Same per SSTable, but far fewer SSTables |
| **Benefit of Bloom filters** | Critical — only way to skip overlapping SSTables | Helpful but less critical; range check already eliminates most |

The Bloom filter implementation (`bloom-filter/bloom_filter.py:37–40`) shows the `__contains__` check that LSM trees typically use to short-circuit lookups:

```python
def __contains__(self, item):
    for pos in _hashes(item, self._k, self._m):
        if not (self._bits[pos // 8] & (1 << (pos % 8))):
            return False
    return True
```

In size-tiered compaction, Bloom filters are practically **mandatory** for acceptable missing-key performance — without them, every SSTable with an overlapping range requires a disk probe. In leveled compaction, the non-overlapping invariant already prunes most candidates, making Bloom filters a nice-to-have optimization rather than a lifeline.

## Why This Matters for Read-Heavy Workloads

The LSM tree in this codebase (`lsm.py`) implements what is essentially a simple size-tiered strategy — SSTables accumulate at a single implicit level and compact all at once. This means the `test_large_dataset` test (`test_lsm.py:108`) with `memtable_threshold=500` and `compaction_threshold=10` can have up to 9 SSTables with fully overlapping ranges at any given time. The assertion `db.get("key_999999") is None` (line 116) — a missing-key lookup — must probe all of them.

A leveled strategy would guarantee that after the L0→L1 compaction, subsequent levels have disjoint ranges, making that same missing-key check touch at most one SSTable per level regardless of data volume.

---

## Topics to Explore

- [function] `sstable-and-compaction/sstable.py:CompactionManager` — Contains both size-tiered and leveled compaction implementations side by side; study how `run_compaction` maintains the non-overlapping invariant for leveled mode
- [function] `log-structured-merge-tree/lsm.py:compact` — The full merge logic starting at line 319 shows how all SSTables are merged into one; compare this "merge everything" approach with the leveled strategy's incremental merges
- [general] `bloom-filter-integration` — The Bloom filter module exists standalone; explore how integrating it into the LSM tree's `get()` path would change the missing-key cost for size-tiered compaction
- [function] `log-structured-merge-tree/lsm.py:get` — Trace the full get() path from memtable through SSTables to understand exactly where overlapping ranges force redundant work
- [general] `write-amplification-tradeoff` — Leveled compaction improves read performance but increases write amplification (each key is rewritten across levels); compare the write path costs that fund the read path savings

## Beliefs

- `lsm-size-tiered-missing-key-probes-all` — In the LSM tree implementation (`lsm.py`), a missing-key lookup must probe every SSTable because the simple compaction strategy allows fully overlapping key ranges across all SSTables
- `leveled-compaction-one-sstable-per-level` — Leveled compaction (`sstable.py`, strategy='leveled') guarantees non-overlapping key ranges within each level, so a missing-key lookup checks at most one SSTable per level
- `bloom-filter-not-integrated-into-lsm` — The Bloom filter module (`bloom_filter.py`) exists independently and is not wired into either LSM tree implementation's get() path, leaving missing-key lookups unoptimized
- `sstable-sparse-index-bounds-per-table-cost` — Both SSTable implementations use a sparse index with binary search (`lsm.py:117`, `sstable.py:200`) to bound the per-SSTable lookup cost, but this doesn't reduce the number of SSTables that must be consulted
- `compaction-threshold-controls-overlap-window` — The `compaction_threshold` parameter (`lsm.py:204`) directly controls how many overlapping SSTables can accumulate before compaction, setting the worst-case missing-key probe count to `threshold - 1`

