# Topic: How reduce-side joins (shuffle + sort + group) differ from these map-side approaches and when each is preferred

**Date:** 2026-05-29
**Time:** 13:30

# Reduce-Side Joins vs. Map-Side Joins

## The Core Difference: Shuffle or No Shuffle

The fundamental distinction comes down to whether you need a **network shuffle** to co-locate records by key.

### Reduce-Side Join (the MapReduce framework)

In `mapreduce-framework/mapreduce.py`, the `MapReduceJob` implements the classic three-phase pipeline. The reducer at line 131 (`_run_reducer`) is where the "join" actually happens in a reduce-side approach:

```python
# Sort by key
all_pairs.sort(key=lambda x: x[0])

# Group by key and reduce
for key, group in groupby(all_pairs, key=lambda x: x[0]):
    values = [v for _, v in group]
    output = self.reducer(key, values)
```

The sequence is:
1. **Map**: Each mapper tags records with the join key and a source identifier (e.g., "users" vs. "orders")
2. **Shuffle**: The framework partitions intermediate output by `hash(key) % num_reducers` (line 104) and writes partition files (`map-{mapper_id}-part-{p}.json`)
3. **Sort + Group**: Each reducer reads all partition files for its slice (line 133–136), sorts by key, and groups with `itertools.groupby`
4. **Reduce**: The reducer receives *all* values for a given key — from *both* datasets — and emits joined records

This is **maximally general**: it works regardless of data size, sort order, or partitioning scheme. But every record from both datasets must be serialized, written to disk, read back, and sorted. That's expensive.

### Map-Side Joins (no shuffle at all)

The three strategies in `map-side-join/map_side_joins.py` each exploit a **precondition** to eliminate the shuffle entirely:

#### 1. Broadcast Hash Join (`BroadcastHashJoin`, line ~68)

**Precondition**: One dataset fits in memory.

The constructor builds a hash table from the small dataset (`_build_hash_table`, line 60), then each mapper independently probes it:

```python
self.hash_table, self.skipped_small = _build_hash_table(small_dataset, small_key)
```

During `join()`, the large dataset is chunked across `num_mappers` (line 83–85), and each mapper does O(1) hash lookups per record. No data moves between mappers — the small dataset is **broadcast** to all of them. This is visible in the stats: `hash_lookups` counts probes, not shuffled records.

#### 2. Partitioned Hash Join (`PartitionedHashJoin`, line ~128)

**Precondition**: Both datasets are already partitioned by the join key (or can be cheaply partitioned).

Both sides are partitioned via `partition_dataset()`, then each partition is joined independently with a local hash table (line 142–145):

```python
left_parts = partition_dataset(left_dataset, self.left_key, self.num_partitions)
right_parts = partition_dataset(right_dataset, self.right_key, self.num_partitions)
```

Each partition only builds a hash table from its own slice of the left side, so memory usage is `O(left_size / num_partitions)` instead of `O(left_size)`. The `_mapper_id` field on output records (line 161) tracks which partition produced each result — confirming partitions are fully independent.

#### 3. Sort-Merge Join (`SortMergeJoin`, referenced in tests)

**Precondition**: Both datasets are sorted by the join key (or sorting is cheap relative to shuffling).

The test at line 192 shows this detects pre-sorted input (`result.stats["sorted_input"] is True`) and avoids re-sorting. When both sides are sorted, a single linear pass merges them — O(n + m) time, no hash table needed.

## When to Prefer Each

| Strategy | Use When | Cost |
|----------|----------|------|
| **Reduce-side** | No preconditions met; arbitrary datasets; need full generality | Full shuffle + sort of both datasets |
| **Broadcast hash** | One side is small enough to fit in memory on every mapper | Memory = small dataset size; no network shuffle |
| **Partitioned hash** | Both sides already co-partitioned by the join key (e.g., output of a prior MapReduce stage) | Memory = partition slice; no cross-partition traffic |
| **Sort-merge** | Both sides already sorted by join key (e.g., SSTable output) | O(n + m) linear scan; no hash table memory |

The reduce-side join in `mapreduce.py` is the **fallback** — it always works because the framework handles co-location via shuffle. Map-side joins are **optimizations** that trade generality for performance by pushing preconditions onto the data layout. The `compare_join_strategies` function (referenced in the test at line 211) exists precisely to make this tradeoff visible by running all three and comparing stats side-by-side.

One thing worth noting: the stream join processor (`stream-join-processor/stream_join_processor.py`) is a completely different paradigm — it operates on unbounded event streams with time windows rather than bounded batch datasets, making it unsuitable for the batch join comparison but relevant for real-time scenarios.

## Topics to Explore

- [function] `map-side-join/map_side_joins.py:SortMergeJoin` — The full sort-merge implementation including how it detects pre-sorted input and handles the merge pass
- [function] `map-side-join/map_side_joins.py:compare_join_strategies` — Runs all three strategies on the same data and returns comparative stats, useful for understanding cost tradeoffs empirically
- [file] `map-side-join/map_side_joins.py` — Lines 200–400 were not included in observations; contains the rest of PartitionedHashJoin, full SortMergeJoin, and utility functions
- [general] `reduce-side-join-implementation` — The codebase has no explicit reduce-side *join* module — worth building one using `MapReduceJob` where the mapper tags records by source and the reducer joins by key, to contrast directly with the map-side implementations
- [file] `stream-join-processor/stream_join_processor.py` — Time-windowed stream joins show a third paradigm beyond batch reduce-side and map-side approaches

## Beliefs

- `broadcast-hash-join-requires-small-side-in-memory` — `BroadcastHashJoin` loads the entire small dataset into a hash table at construction time, so the small side must fit in a single mapper's memory
- `partitioned-hash-join-partitions-both-sides` — `PartitionedHashJoin.join()` calls `partition_dataset()` on both inputs before joining, meaning partitions are computed at join time rather than assumed from storage layout
- `mapreduce-shuffle-uses-hash-partitioning` — The MapReduce framework assigns intermediate key-value pairs to reducers via `hash(k) % self.num_reducers` (mapreduce.py line 104), which is the same partitioning function that makes partitioned hash joins work
- `map-side-joins-track-mapper-id-on-output` — All three map-side join strategies attach a `_mapper_id` field to each output record, enabling verification that partitions/mappers operated independently
- `reduce-side-sort-group-uses-itertools-groupby` — The reducer in `mapreduce.py` relies on `itertools.groupby` over pre-sorted pairs (line 141–142), meaning records must be sorted by key before grouping — the sort step is not optional

