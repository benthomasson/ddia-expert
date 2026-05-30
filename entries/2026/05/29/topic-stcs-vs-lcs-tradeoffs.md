# Topic: DDIA Chapter 3 covers why STCS favors write throughput while LCS bounds read amplification, and when each is appropriate (Cassandra uses STCS by default, RocksDB uses leveled)

**Date:** 2026-05-29
**Time:** 13:24

# STCS vs. LCS: Compaction Strategy Tradeoffs in This Codebase

## Two Implementations, Two Philosophies

This repo has two LSM-tree modules that illustrate the compaction spectrum DDIA Chapter 3 describes:

1. **`log-structured-merge-tree/lsm.py`** — A basic LSM tree with a single, threshold-based compaction (effectively STCS-like behavior)
2. **`sstable-and-compaction/sstable.py`** — A full implementation with both **size-tiered** and **leveled** strategies, selectable via `CompactionManager`

## Size-Tiered Compaction (STCS)

In `sstable-and-compaction/test_sstable.py:47`, you can see STCS in action:

```python
manager = CompactionManager(tmpdir, strategy='size_tiered', min_threshold=2)
```

The simpler LSM tree in `lsm.py` demonstrates the same idea at line 316:

```python
if len(self._sstables) >= self._compaction_threshold:
```

When the count of SSTables hits a threshold, they all get merged together. This is the essence of size-tiered: SSTables accumulate at the same "tier" (roughly similar size), and when enough pile up, they merge into a bigger one.

**Why STCS favors write throughput:** Each write only goes to the memtable (`lsm.py:232`, `self._memtable[key] = value`), then gets flushed to a new SSTable when the memtable fills (`lsm.py:244`, `if len(self._memtable) >= self._threshold`). Compaction happens lazily — you can let many SSTables accumulate before merging. This means writes never stall waiting for compaction to rearrange levels. The write path is: memtable → flush → done. Compaction is deferred background work.

**The read cost:** Notice `test_lsm.py:59-65` — with `compaction_threshold=100`, the test creates multiple SSTables with the same key `"k"` and must search newest-to-oldest to find the right value. Without compaction, a read might check the memtable, then every SSTable in reverse order. That's **read amplification** — potentially O(N) SSTables scanned per lookup.

## Leveled Compaction (LCS)

In `sstable-and-compaction/test_sstable.py:108`:

```python
mgr = CompactionManager(lcs_dir, strategy='leveled', l0_compaction_trigger=2)
```

The leveled strategy organizes SSTables into levels with size constraints. From the grep results, we can see the key parameters in `sstable.py`:

- **`level_base_size`** (line 307): 10MB default — the target size for Level 1
- **`max_levels`** (line 309): 7 levels
- **`_level_max_size`** (line 377-380): Each level is `base_size * fanout^(level-1)` — exponential growth

The compaction trigger logic (lines 383-388) checks two conditions:
1. Level 0 has too many SSTables (≥ `l0_trigger`)
2. Any level exceeds its size budget

After compaction, SSTables are promoted: `test_sstable.py:119` asserts `result[0].level == 1`.

**Why LCS bounds read amplification:** In a leveled scheme, each level (except L0) has **non-overlapping key ranges**. A point read needs to check at most one SSTable per level. With 7 levels, that's at most ~7 SSTables checked — a bounded O(log N) regardless of how much data you have.

**The write cost:** Leveled compaction is more aggressive — when a level overflows, its SSTables get merge-sorted into the next level, potentially rewriting SSTables that already exist there. A single write can trigger a cascade of rewrites across levels. This is **write amplification** — the same data gets rewritten many times as it moves down levels.

## The Tradeoff Matrix

| | STCS | LCS |
|---|---|---|
| **Write amp** | Low (merge similar-size groups lazily) | High (cascade rewrites across levels) |
| **Read amp** | High (scan many unsorted runs) | Low (≤1 SSTable per level) |
| **Space amp** | High (duplicate keys across tiers) | Low (compaction deduplicates aggressively) |
| **Best for** | Write-heavy (logging, time-series) | Read-heavy (user-facing queries) |

This is why Cassandra defaults to STCS (optimized for its common write-heavy workloads) while RocksDB defaults to leveled (optimized for serving reads in embedded database scenarios like MyRocks or CockroachDB's storage layer).

## What's Missing

The observations don't include the full `CompactionManager` implementation (lines 250–438 of `sstable.py` were truncated). The actual merge logic for leveled compaction — specifically how it selects which L(N) SSTable overlaps with L(N+1) and does a targeted merge — would show the key-range non-overlap invariant that makes LCS reads efficient. The `_get_levels` method (line 371) and `_level_max_size` (line 377) are visible, but the actual merge-selection and output-splitting logic is not.

---

## Topics to Explore

- [function] `sstable-and-compaction/sstable.py:CompactionManager` — The full compaction manager contains the strategy selection logic, merge-triggering heuristics, and level-promotion code that enforces the non-overlapping invariant in LCS
- [function] `log-structured-merge-tree/lsm.py:compact` — The simple all-at-once merge (line 340+) shows tombstone removal during compaction and how the single-run merge differs from leveled's targeted approach
- [function] `sstable-and-compaction/sstable.py:merge_sstables` — The k-way merge of multiple SSTable readers is the core primitive both strategies depend on; understanding the heap-based merge illuminates why sorted runs are essential
- [general] `bloom-filter-integration` — DDIA explains that Bloom filters mitigate STCS's read amplification by skipping SSTables that definitely don't contain a key; the `bloom-filter/` module in this repo implements exactly that optimization
- [file] `log-structured-merge-tree/lsm.py` — Compare the WAL→memtable→flush write path (lines 228-244) with the SSTable read path (searching newest-first) to see write/read amplification firsthand

## Beliefs

- `stcs-compaction-is-count-triggered` — Size-tiered compaction in `lsm.py` triggers when the number of SSTables reaches `compaction_threshold`, not based on SSTable sizes or level overflow
- `lcs-levels-grow-exponentially` — Leveled compaction sizes each level as `level_base_size * fanout^(level-1)` (`sstable.py:380`), meaning Level 2 is 10x Level 1 by default
- `lcs-l0-is-special` — Level 0 in leveled compaction triggers based on SSTable count (`l0_compaction_trigger`), not total size, because L0 SSTables can have overlapping key ranges unlike higher levels
- `newest-sstable-wins-reads` — Both implementations search SSTables from newest to oldest (highest sequence number first), so the first match found is authoritative; this is what makes deferred compaction correct despite duplicate keys across SSTables
- `compaction-removes-tombstones` — Tombstones (`TOMBSTONE = b""` in `lsm.py:10`, `TOMBSTONE_MARKER = 0xFF` in `sstable.py:12`) are only garbage-collected during compaction, not at read time — until then they must persist to shadow deleted keys in older SSTables

