# Topic: LevelDB's `table/merger.cc` implements the canonical heap-based k-way merge; comparing it to this Python impl reveals what production correctness requires (direction reversal, snapshot reads, seek optimization)

**Date:** 2026-05-29
**Time:** 07:32

I'll work with the observations already gathered.

---

# K-Way Merge: LevelDB's `merger.cc` vs. This Python LSM Implementation

## What the Python Implementation Does

The LSM tree in `log-structured-merge-tree/lsm.py` uses two distinct merge strategies, both simpler than LevelDB's approach:

**1. Range scan merge (lines 275–298)** — Collects everything into a dictionary, using priority to resolve duplicates:

```python
merged: dict = {}  # key -> (priority, value_bytes)
```

It walks SSTables, immutable memtables, and the active memtable in order, assigning increasing priority so that newer sources override older ones. Then it sorts the keys and emits results. This is a **materialize-everything** approach — it loads all entries into memory before producing output.

**2. Compaction merge (lines 323–347)** — Uses `heapq` to merge SSTable iterators:

```python
# k-way merge: iterate all SSTables, newest wins
```

It tracks `prev_key` (line 335) to skip duplicates, keeping only the first occurrence (from the newest SSTable). This is closer to LevelDB's approach but still fundamentally simpler.

## What LevelDB's `merger.cc` Adds — and Why It Matters

LevelDB's `MergingIterator` is a production-grade k-way merge that addresses three problems this Python implementation doesn't:

### 1. Direction Reversal

LevelDB's merging iterator supports both `Next()` and `Prev()`, maintaining a **direction state** that tracks which way the heap is ordered. When you call `Prev()` after calling `Next()`, it must re-seek all child iterators to the correct position. The Python implementation has no concept of this — `scan()` (line 150) and `scan_all()` (line 167) are strictly forward-only. The grep results confirm this: the only `reverse` usage in lsm.py is `reversed(self._sstables)` (line 259) and `reversed(self._immutable_memtables)` (line 254) — these reverse the **priority order** of sources, not the iteration direction within them.

This matters because real databases need reverse iteration for `ORDER BY DESC` queries, cursor-based pagination backward, and range scans from high to low.

### 2. Snapshot Reads

LevelDB's merge iterator operates within a snapshot context — it only surfaces entries visible at a particular sequence number. The Python implementation has no sequence numbers or snapshots at all. During the range scan merge (lines 275–298), it reads the current state of all sources with no isolation guarantee. If a flush happens mid-scan (the memtable threshold is reached during iteration), the results could be inconsistent.

The `sstable-and-compaction/sstable.py` implementation is slightly more aware — its `SSTableEntry` dataclass (line 27) carries a `timestamp` field, which *could* be used for snapshot filtering. But the merge logic in `lsm.py` uses only structural priority (which source is newer), not per-entry timestamps.

For contrast, the `write-skew-detection/ssi_database.py` (lines 59–80) shows what proper snapshot reads look like: `_visible_value()` filters entries by `snapshot_ts`, returning only versions committed before the reader's snapshot.

### 3. Seek Optimization

LevelDB's `MergingIterator::Seek(target)` calls `Seek(target)` on every child iterator, then rebuilds the heap. This is O(k log k) where k is the number of children. The Python SSTable's `scan()` (line 150) uses the sparse index to find a starting position within a single file, but the range scan merge (lines 275–298) doesn't use `Seek` at all — it scans all entries from all SSTables and filters in memory. For a range scan over keys "z0" to "z9" in a database with millions of entries starting with "a", this reads the entire database.

The sparse index lookup in SSTable.scan (lines 155–159) is the right idea:
```python
keys = [k for k, _ in self._sparse_index]
idx = bisect.bisect_right(keys, start_key) - 1
```

But this optimization lives inside individual SSTables, not at the merge level. LevelDB pushes `Seek` down to each child iterator *and* coordinates the heap rebuild at the merge level.

## The Deeper Lesson

The Python implementation is **correct for its scope** — single-threaded, forward-only, full-scan workloads. The tests in `test_lsm.py` confirm the semantics: `test_range_scan_across_sources` (line 154) verifies that newer memtable entries override SSTable entries, and `test_multiple_sstables_newest_wins` (line 60) checks cross-SSTable priority. These invariants hold.

But LevelDB's `merger.cc` reveals that a production merge iterator is fundamentally a **bidirectional, seekable, snapshot-aware** abstraction — not just a heap. The heap is the data structure; the *interface contract* (forward, backward, seek, snapshot isolation) is what makes it production-ready. This Python implementation captures the algorithm without the interface.

---

## Topics to Explore

- [general] `leveldb-merger.cc` — Read LevelDB's actual `table/merger.cc` source to see the direction enum, `FindSmallest`/`FindLargest` helper methods, and how it rebuilds the heap on direction change
- [function] `log-structured-merge-tree/lsm.py:range_scan` — The full range_scan method (around lines 265–300) to see how the priority-based dict merge works and why it doesn't scale to large datasets
- [file] `sstable-and-compaction/sstable.py` — The compaction merge logic (lines 200+) likely uses `heapq.merge` with timestamps, showing a more sophisticated merge than the LSM tree's dict approach
- [file] `write-skew-detection/ssi_database.py` — The `_visible_value` and `_snapshot` methods (lines 59–80) demonstrate the snapshot isolation that a production merge iterator would need
- [general] `rocksdb-merge-operator` — RocksDB extends LevelDB's merge with a `MergeOperator` that supports partial merges during compaction — the next step beyond what both implementations show

## Beliefs

- `lsm-range-scan-materializes-all` — `LSMTree.range_scan` (lsm.py ~lines 275–298) loads all matching entries from all SSTables into a dict before returning, making it O(total entries) regardless of range size
- `lsm-no-snapshot-isolation` — The LSM tree implementation has no sequence numbers or snapshot mechanism; concurrent flushes during a range scan can produce inconsistent reads
- `lsm-forward-only-iteration` — SSTable iterators (`scan`, `scan_all`) support only forward iteration; there is no `Prev()` or reverse scan capability
- `sstable-has-timestamps-lsm-ignores-them` — `SSTableEntry` in `sstable.py` carries a `timestamp` field, but `lsm.py`'s SSTable format uses only structural ordering (SSTable sequence number) for conflict resolution
- `lsm-compaction-uses-heap-merge` — Compaction (lsm.py ~line 323) uses `heapq`-based k-way merge with `prev_key` deduplication, while range scan uses a simpler dict-based merge — two different strategies in the same codebase

