# Topic: `hash(k) % num_reducers` can produce skew with non-uniform key distributions; contrast with the range-partitioning implementation in this repo

**Date:** 2026-05-29
**Time:** 13:33

## Hash-Mod Partitioning vs. Range Partitioning

### The skew problem with `hash(k) % num_reducers`

In `mapreduce-framework/mapreduce.py:109`, the mapper assigns each intermediate key to a reducer partition:

```python
p = hash(k) % self.num_reducers
```

This is a **fixed, static assignment**. The number of reducers is set at job creation (`num_reducers: int = 2`, line 33) and never changes. Every key deterministically maps to exactly one partition for the entire job.

The skew problem: if a small number of keys dominate the dataset (e.g., a word-count where "the" appears millions of times), **all records for that key land on a single reducer**. Hash-mod distributes *distinct keys* roughly evenly across partitions, but it cannot split a *single hot key* across multiple reducers. One reducer gets overwhelmed while others sit idle — this is the "straggler" problem from DDIA Chapter 6.

The same pattern appears across the repo — `secondary-index-partitioning/secondary_index_partitioning.py:56` uses `hash(pk) % self.num_partitions`, and `partitioned-log/partitioned_log.py:149` uses `h % topic_obj.num_partitions`. All share the same vulnerability: partition count is fixed at configuration time, and hot keys create hotspots.

### How range partitioning adapts

`range-partitioning/range_partitioning.py` takes a fundamentally different approach. Instead of a hash function choosing a fixed slot, `RangePartitionedStore` maintains a sorted list of partition boundaries (`_boundaries`, line 111) and routes keys via binary search:

```python
def _find_partition_index(self, key: str) -> int:
    idx = bisect.bisect_right(self._boundaries, key) - 1
    return idx
```

The critical difference is **dynamic splitting** (`range_partitioning.py:118-122`). When a partition exceeds `max_partition_size`, it splits at the median key:

```python
if partition.size > self.max_partition_size:
    new_right = partition.split()
    self._partitions.insert(idx + 1, new_right)
    self._boundaries.insert(idx + 1, new_right.start_key)
```

The `Partition.split()` method (line 68) picks the median of the *actual stored keys*, so the two resulting partitions are roughly equal in size — regardless of key distribution. If a particular key range is hot, it keeps splitting until each partition is below the threshold. This is the same principle as B-tree page splits.

### The tradeoff

| Property | Hash-mod (`mapreduce.py`) | Range (`range_partitioning.py`) |
|---|---|---|
| Partition count | Fixed at job start | Dynamic, grows/shrinks with data |
| Hot key handling | Cannot split; straggler risk | Splits hot ranges at median |
| Key ordering | Destroyed by hash | Preserved; enables `range_scan` |
| Range queries | Requires scanning all partitions | Routes to relevant partitions via `bisect` |
| Rebalancing | Requires re-hashing everything | `merge_small_partitions()` handles shrinkage |

Range partitioning enables efficient range scans (`range_scan`, line 137) that cross partition boundaries seamlessly — something hash-mod partitioning fundamentally cannot do without a scatter-gather across all reducers. The cost is more complex routing and bookkeeping for boundaries.

Note that range partitioning has its *own* skew risk: if keys arrive in sorted order, all writes hit the rightmost partition. The median-split strategy in `Partition.split()` mitigates this for data already stored, but doesn't prevent temporary hotspots during sequential ingestion.

## Topics to Explore

- [function] `range-partitioning/range_partitioning.py:Partition.split` — The median-key split strategy and how it guarantees balanced halves; compare with B-tree page splits in the btree module
- [file] `secondary-index-partitioning/secondary_index_partitioning.py` — Uses hash-mod for both document-partitioned and term-partitioned secondary indexes; shows two different scatter/gather tradeoffs on top of the same hash scheme
- [file] `consistent-hashing/consistent_hashing.py` — A third partitioning strategy that handles node addition/removal with minimal key movement, solving a different problem than either hash-mod or range
- [function] `mapreduce-framework/mapreduce.py:_apply_combiner` — The combiner is the MapReduce framework's partial mitigation for skew: reducing data volume *before* shuffle, though it doesn't eliminate the single-reducer bottleneck for hot keys
- [general] `partition-rebalancing-strategies` — How the merge path (`merge_small_partitions`) interacts with split to keep partition counts stable over mixed insert/delete workloads

## Beliefs

- `hash-mod-partition-count-is-static` — In `mapreduce.py`, `num_reducers` is fixed at job creation and never changes during execution; all partition assignments are determined by `hash(k) % num_reducers` with no rebalancing
- `range-partition-splits-at-median` — `Partition.split()` divides at the median index of stored keys, guaranteeing the two resulting partitions differ in size by at most one element
- `range-partition-boundaries-are-contiguous` — `RangePartitionedStore` maintains the invariant that the first partition starts at `""`, the last has `end_key=None`, and each partition's `end_key` equals the next partition's `start_key`
- `hash-mod-destroys-key-order` — Hash-mod partitioning scatters lexicographically adjacent keys across different partitions, making range queries require a full scatter-gather across all reducers (the MapReduce `run()` method re-sorts the final results at line 79)
- `range-partition-merge-is-greedy-left-to-right` — `merge_small_partitions` sweeps left-to-right and greedily merges adjacent pairs whose combined size is at or below `min_partition_size`, which can produce different results than an optimal merge strategy

