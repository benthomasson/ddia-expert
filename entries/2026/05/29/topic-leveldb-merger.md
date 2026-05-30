# Topic: Read LevelDB's actual `table/merger.cc` source to see the direction enum, `FindSmallest`/`FindLargest` helper methods, and how it rebuilds the heap on direction change

**Date:** 2026-05-29
**Time:** 10:12

# LevelDB's `merger.cc` — Direction Enum, Heap Helpers, and Direction-Change Rebuilds

## What the Observations Show

**The observations are insufficient to answer this question directly.** The target codebase (`ddia-implementations`) does not contain LevelDB's C++ source — there is no `table/merger.cc`, no `MergeIterator` class with a direction enum, and the grep for `FindSmallest`/`FindLargest` returned zero matches. This is a Python reference implementation repo, not a fork of LevelDB.

However, the codebase *does* contain two merge implementations that illustrate the same core concepts LevelDB's merger solves — and notably, they both skip the direction-tracking optimization that makes LevelDB's version interesting.

## What This Codebase Does Instead

### Simple heap merge in `sstable-and-compaction/sstable.py`

The k-way merge at lines 253–276 uses Python's `heapq` — a min-heap — for forward-only iteration:

```python
heap: list = []                                                    # line 253
heapq.heappush(heap, (entry.key, -entry.timestamp, i, entry, it)) # line 261
key, neg_ts, idx, entry, it = heapq.heappop(heap)                 # line 276
```

The tuple `(key, -entry.timestamp, i, ...)` ensures that when two SSTables contain the same key, the **newest timestamp wins** (negated so the min-heap picks the largest timestamp). The third element `i` is the source index, used as a tiebreaker to avoid comparing incomparable `SSTableEntry` objects.

This merge is **forward-only** — it never reverses direction. There is no direction enum, no `FindLargest`, and no heap rebuild.

### LSM tree merge in `log-structured-merge-tree/lsm.py`

At line 323, the comment reads: `# k-way merge: iterate all SSTables, newest wins`. The same pattern — `heapq` for forward iteration, newest-timestamp-wins deduplication. Again, forward-only.

## What LevelDB's `merger.cc` Actually Does (and Why It Matters)

The gap between these reference implementations and LevelDB's real merger is exactly the point worth understanding:

1. **Direction enum (`kForward`/`kReverse`)**: LevelDB's `MergingIterator` tracks which direction it's currently iterating. This is necessary because LevelDB supports bidirectional iteration (`Next()`, `Prev()`, `Seek()`, `SeekToLast()`).

2. **`FindSmallest()` / `FindLargest()`**: These are O(n) linear scans over the child iterators to find which one currently points to the smallest (or largest) key. They're called when rebuilding the heap after a direction change or `Seek()`.

3. **Heap rebuild on direction change**: When you call `Prev()` after a series of `Next()` calls (or vice versa), all child iterators must be repositioned. Each child's cursor may be pointing past the current merge position in the old direction. The merger calls `Prev()`/`Next()` on every child iterator, then rebuilds the heap from scratch via `FindSmallest()` or `FindLargest()`.

The reference implementations here avoid this complexity entirely by only supporting forward iteration during compaction — a valid simplification since compaction never needs reverse traversal. But any storage engine that exposes bidirectional range scans to users (as LevelDB does via its public `Iterator` interface) must handle direction changes.

## What's Missing

To actually read and explain `merger.cc`, you would need:
- The LevelDB source tree (typically at `table/merger.cc` and `table/merger.h`)
- Or a local clone of `google/leveldb` from GitHub

The `ddia-implementations` repo implements the *concepts* from DDIA (SSTables, LSM trees, compaction) but does not replicate LevelDB's bidirectional merge iterator machinery.

## Topics to Explore

- [file] `sstable-and-compaction/sstable.py:253` — The k-way merge heap with timestamp-based deduplication; compare against what a bidirectional version would require
- [function] `sstable-and-compaction/sstable.py:_push_next` — Helper at line 265 that advances one child iterator and pushes its next entry onto the heap — the forward-only analog of LevelDB's direction-aware advancement
- [file] `log-structured-merge-tree/lsm.py` — Full LSM tree implementation; check whether `scan()` supports reverse iteration or is forward-only
- [general] `leveldb-merger-cc-source` — Clone `google/leveldb` and read `table/merger.cc` directly to see the `Direction` enum, `FindSmallest`/`FindLargest`, and the `SwitchDirection` logic
- [general] `bidirectional-merge-iterator` — Consider implementing a bidirectional merge iterator in this repo that tracks direction and rebuilds the heap on reversal, filling the gap between these reference implementations and LevelDB's real design

## Beliefs

- `sstable-merge-forward-only` — The k-way merge in `sstable-and-compaction/sstable.py` is forward-only with no direction tracking or reverse iteration support
- `heap-tuple-ordering-for-dedup` — The merge heap uses `(key, -timestamp, source_idx)` tuples so that min-heap pop yields the smallest key with the newest timestamp, implementing last-writer-wins deduplication
- `no-leveldb-source-in-repo` — The `ddia-implementations` repo contains no LevelDB C++ source; `FindSmallest`, `FindLargest`, and direction enums are absent from the codebase
- `compaction-avoids-direction-complexity` — Because compaction only needs forward iteration, both merge implementations avoid the heap-rebuild-on-direction-change logic that LevelDB's `MergingIterator` requires for its public bidirectional `Iterator` API

