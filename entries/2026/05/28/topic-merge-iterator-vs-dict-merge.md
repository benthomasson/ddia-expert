# Topic: Production LSM trees (LevelDB, RocksDB) use a heap-based merge iterator instead of materializing all results into a dict; explore the memory and latency tradeoffs

**Date:** 2026-05-28
**Time:** 19:08

No problem — the observations have enough context to give a thorough explanation.

---

# Merge Strategy: Dict Materialization vs. Heap-Based Merge Iterator

## What the codebase does today

The LSM tree in `log-structured-merge-tree/lsm.py` uses a **dict-materialization** approach for both range scans and compaction. Look at lines 274–298 for `range_scan`:

```python
merged: dict = {}  # key -> (priority, value_bytes)   # line 276
priority = 0
# ... iterates all SSTables oldest-to-newest ...
    merged[k] = (priority, v)                           # line 282
# ... then immutable memtables ...
    merged[k] = (priority, mt[k])                       # line 288
# ... then the active memtable ...
    merged[k] = (priority, self._memtable[k])           # line 293

for k in sorted(merged):                                # line 297
    _, v = merged[k]                                    # line 298
```

And again at lines 323–347 for `compact`:

```python
# k-way merge: iterate all SSTables, newest wins       # line 323
merged: List[Tuple[str, bytes]] = []                    # line 334
# ... collects everything, then writes ...
new_sst = SSTable.write(path, seq, merged, ...)         # line 347
```

Both methods **load every key-value pair into an in-memory data structure** before producing output. The dict approach is simple and correct — newer values overwrite older ones by priority, and `sorted(merged)` produces the final order. But it has significant tradeoffs that production systems avoid.

## Why production systems use a heap-based merge iterator

### Memory: O(total data) vs. O(number of sources)

The dict approach in `lsm.py:276` allocates memory proportional to the **total number of unique keys across all sources**. For a range scan over 10 SSTables each containing 1M entries, you might materialize millions of key-value pairs into a Python dict before returning the first result.

A heap-based merge iterator (a min-heap of size `k`, where `k` = number of sources) holds **at most one entry per source** at any time. For the same 10-SSTable scan, the heap contains exactly 10 entries. Memory is O(k), not O(n). This is the pattern `heapq` (imported at `lsm.py:6` but unused for range scans) is designed for.

LevelDB's `MergingIterator` and RocksDB's `MergingIterator` both work this way: each child iterator (one per SSTable + one for the memtable) advances independently, and the heap always yields the globally smallest key.

### Latency: full materialization vs. streaming

The dict approach has **high first-result latency**. A `range_scan("a", "b")` must scan every SSTable's `[a, b)` range, insert all results into the dict, sort, and *then* return. If the caller only needs the first 10 results (a common pattern with `LIMIT` queries), all that work is wasted.

A heap merge iterator is **streaming**: it yields the first result after just one `heappush` per source and a single `heappop`. Each subsequent result costs O(log k) for the heap operation. This gives:

| Property | Dict (this impl) | Heap merge iterator |
|---|---|---|
| First result latency | O(n log n) — must scan + sort all | O(k log k) — one push per source |
| Memory | O(n) — all unique keys | O(k) — one entry per source |
| Early termination | No benefit | Stops I/O immediately |
| Complexity | Simple | Must handle duplicate keys across sources |

where *n* = total entries in range, *k* = number of sources (SSTables + memtables).

### How a heap merge iterator handles "newest wins"

The tricky part the dict solves so naturally — newer values overwriting older ones at `lsm.py:282-293` — requires more care with a heap. The standard approach:

1. Push `(key, sequence_number, source_id, value)` onto the heap. The sequence number breaks ties: **higher sequence = newer**.
2. Pop the minimum key. If the next entry on the heap has the **same key**, skip it (it's an older version). Keep popping duplicates.
3. Yield the winning entry.

The `sstable-and-compaction/sstable.py` file uses timestamps for this purpose (`SSTableEntry.timestamp` at line 28), which serve the same role as sequence numbers. Notice it already imports `heapq` at line 2 — this module is better positioned for a proper merge iterator, though the observation window didn't show whether it's used for range scans or only for compaction.

### The compaction case

For compaction (`lsm.py:323-347`), the tradeoff is slightly different. Compaction *must* read all entries regardless — there's no early termination benefit. But the memory advantage still matters: compacting a 10 GB level shouldn't require 10 GB of RAM. Production systems stream the merge directly into the output SSTable, holding O(k) memory throughout. The current implementation at line 334 (`merged: List[...]`) buffers everything, which would fail on datasets larger than available memory.

## What `heapq` is actually used for

Despite importing `heapq` at `lsm.py:6`, the implementation doesn't use it for the merge operations that would benefit most. The grep results show the merge logic uses dicts and lists instead (lines 276, 334). This is a clear gap between the reference implementation and production LSM trees — and an excellent area for improvement if you want to bring this implementation closer to LevelDB/RocksDB behavior.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:range_scan` — The dict-based merge at lines 274–298; refactoring to a `heapq`-based streaming iterator would be a great exercise
- [function] `sstable-and-compaction/sstable.py:SSTableReader.get` — Uses binary search on the sparse index (line 200+), which is the per-SSTable lookup that a merge iterator would call `next()` on
- [general] `leveldb-merging-iterator` — LevelDB's `table/merger.cc` implements the canonical heap-based k-way merge; comparing it to this Python impl reveals what production correctness requires (direction reversal, snapshot reads, seek optimization)
- [function] `log-structured-merge-tree/lsm.py:compact` — The compaction merge at lines 323–347 materializes all entries; converting to streaming would demonstrate O(k) memory compaction
- [general] `bloom-filter-optimization` — Production LSMs avoid touching SSTables entirely for missing keys using bloom filters; this complements the merge iterator by reducing k before the heap is even built

## Beliefs

- `lsm-range-scan-materializes-all` — `lsm.py:range_scan` loads all matching entries from every SSTable and memtable into a dict before returning any results, making first-result latency proportional to total result set size
- `heapq-imported-but-unused-for-merge` — `lsm.py` imports `heapq` (line 6) but uses dict-based merging for range scans and list-based merging for compaction; no heap-based k-way merge is implemented
- `sstable-scan-is-streaming` — `SSTable.scan()` and `scan_all()` use Python generators (yield), meaning individual SSTables already support streaming — only the cross-SSTable merge layer materializes
- `compaction-buffers-all-entries` — `lsm.py:compact` collects all merged entries into a list (line 334) before writing the output SSTable, requiring O(n) memory proportional to total data rather than O(k) proportional to number of input SSTables
- `newer-overwrites-via-priority-counter` — The range scan merge uses an incrementing priority counter (lines 277–293) where higher priority = newer source; the dict naturally deduplicates by overwriting older entries for the same key

