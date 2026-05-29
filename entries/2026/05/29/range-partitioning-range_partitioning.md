# File: range-partitioning/range_partitioning.py

**Date:** 2026-05-29
**Time:** 09:03

## Purpose

This file implements **range-based partitioning** — the technique described in DDIA Chapter 6 where a keyspace is divided into contiguous, non-overlapping ranges, each assigned to a partition. It owns two responsibilities: storing sorted key-value data within individual partitions (`Partition`), and routing operations to the correct partition plus managing partition lifecycle — splits and merges — (`RangePartitionedStore`).

This is the counterpart to the `consistent-hashing` implementation in the same repo. Where hash partitioning distributes keys pseudo-randomly for uniform load, range partitioning preserves key ordering, which makes range scans efficient but creates hot-spot risk on sequential writes.

## Key Components

### `Partition`

A single shard holding key-value pairs in **sorted order** within a half-open range `[start_key, end_key)`. `end_key=None` means unbounded to the right.

- **`put(key, value)`** — Upsert via `bisect_left`. Finds the insertion point, overwrites if the key exists, otherwise inserts in-place. O(n) due to list insertion, but keeps the list sorted at all times.
- **`get(key)`** — O(log n) lookup via binary search.
- **`delete(key)`** — O(n) removal (list pop), returns a boolean indicating whether the key existed.
- **`range_scan(start, end)`** — Returns all pairs in `[start, end)` using two binary searches to find the slice boundaries. This is where range partitioning pays off — a scan within one partition is a single slice operation.
- **`contains_key(key)`** — Tests range membership, not data membership. Returns `True` if the key falls within `[start_key, end_key)` regardless of whether data exists for it.
- **`split()`** — Splits at the **median index** (not median value). Mutates `self` to keep the left half and returns a new `Partition` for the right half. Updates `self.end_key` to the median key.
- **`merge(other)`** — Absorbs an adjacent partition by extending keys/values and adopting the other's `end_key`. Assumes `other` is the immediate right neighbor.

### `PartitionInfo`

A read-only dataclass snapshot of partition metadata — used by `get_partition_info()` and `get_partition_for_key()` to expose partition state without leaking mutable internals.

### `RangePartitionedStore`

The coordinator that maintains the partition list and routes operations.

- **`_boundaries`** — A sorted list of partition start keys, kept in lockstep with `_partitions`. This is the routing table: `bisect_right(boundaries, key) - 1` gives the owning partition index in O(log p) where p is the partition count.
- **`put(key, value)`** — Routes to the correct partition, then checks if `size > max_partition_size` and triggers an automatic split. The split inserts the new partition and its boundary key into the parallel lists.
- **`range_scan(start_key, end_key)`** — Finds the first and last relevant partitions via `_find_partition_index`, then iterates through all partitions in that span, collecting results. Handles the edge case where `end_key` exactly matches a boundary (line 119–120) to avoid including an empty scan of the next partition.
- **`merge_small_partitions()`** — Greedy left-to-right sweep. If two adjacent partitions have a combined size ≤ `min_partition_size`, the left absorbs the right. After a merge, re-checks the same index (doesn't increment `i`) to allow cascading merges.

## Patterns

**Parallel arrays for routing.** `_partitions` and `_boundaries` are kept in sync — index `i` in both refers to the same partition. This is a simple alternative to a balanced tree of partition metadata; it trades O(n) insert/delete on the lists for O(1) indexed access and O(log n) binary search routing.

**Median-index split.** The split point is the key at position `len // 2`, not the lexicographic midpoint of the range. This produces roughly equal-sized partitions by count but can produce unequal ranges if keys are clustered.

**Half-open intervals `[start, end)`.** The last partition uses `end_key=None` as an open upper bound. Every key belongs to exactly one partition — no gaps, no overlaps.

**Automatic split, manual merge.** Splits are triggered inline during `put()`, but merges require an explicit call to `merge_small_partitions()`. This asymmetry is typical in real systems (e.g., HBase, CockroachDB) where splits are urgent (a partition is too large to serve) but merges can be deferred to a background process.

## Dependencies

**Imports:** Standard library only — `bisect` for binary search, `uuid` for partition IDs, `dataclass` and `typing` for structure.

**Imported by:** `test_range_partitioning.py` and `tester_test_range_partitioning.py` — the test suite and its validation harness.

## Flow

A typical write path:

1. `RangePartitionedStore.put("customer:42", data)` is called.
2. `_find_partition_index` does `bisect_right(self._boundaries, "customer:42") - 1` to find partition index.
3. The partition's `put()` does `bisect_left` to find the sorted insertion point, inserts or overwrites.
4. If the partition now exceeds `max_partition_size`, `split()` is called — the partition divides its data at the median, a new right partition is created, and `_partitions`/`_boundaries` are updated.

A typical range scan:

1. `range_scan("customer:10", "customer:50")` identifies the start and end partition indices.
2. Iterates partitions from start to end index, calling each partition's `range_scan` with the same bounds.
3. Each partition does two `bisect_left` calls to slice its sorted lists, returns the matching pairs.
4. Results are concatenated in partition order, producing a globally sorted result.

## Invariants

- **Partitions are contiguous and non-overlapping.** `_partitions[i].end_key == _partitions[i+1].start_key` always holds. Gaps or overlaps would cause lost or duplicated data.
- **`_boundaries` is always sorted and matches `_partitions`.** `_boundaries[i] == _partitions[i].start_key` for all `i`.
- **Keys within a partition are sorted.** Every mutation (`put`, `delete`, `split`, `merge`) preserves sorted order of `_keys`.
- **The first partition always starts at `""`** and the last partition always has `end_key=None`. Together they cover the entire string keyspace.
- **Split requires ≥1 key.** `split()` indexes `_keys[len//2]` without checking — calling it on an empty partition would raise `IndexError`.
- **Merge assumes adjacency.** `merge(other)` extends `self._keys` with `other._keys` without re-sorting. If `other` is not the immediate right neighbor, the sorted invariant breaks silently.

## Error Handling

Minimal. The code does not validate inputs or guard against misuse:

- No check that keys are strings. Passing non-string keys would produce undefined comparison behavior.
- `split()` on an empty partition raises `IndexError` — but this can't happen through the normal API since splits only trigger when `size > max_partition_size`.
- `merge()` trusts that the caller passes the adjacent right partition. No adjacency validation.
- `delete()` and `get()` on missing keys return `False`/`None` respectively — no exceptions.
- No thread safety. Concurrent access to any `put`/`delete`/`split`/`merge` path would corrupt the sorted lists.

## Topics to Explore

- [file] `consistent-hashing/consistent_hashing.py` — The hash-based counterpart; compare trade-offs between range and hash partitioning (scan efficiency vs. hot-spot avoidance)
- [file] `range-partitioning/test_range_partitioning.py` — See how split/merge behavior is exercised and what edge cases the tests cover
- [function] `range-partitioning/range_partitioning.py:range_scan` — The boundary-crossing logic at lines 113–125 is the trickiest part; trace through a scan that spans 3+ partitions
- [general] `ddia-chapter-6-partitioning` — DDIA Chapter 6 covers the theory behind partition strategies, rebalancing, and request routing that this code implements
- [file] `secondary-index-partitioning/secondary_index_partitioning.py` — How secondary indexes interact with range-partitioned primary data; a natural extension of this module

## Beliefs

- `range-partitioning-split-at-median-index` — `Partition.split()` divides at `len(keys)//2` (count-based median), not the lexicographic midpoint of the key range, so splits produce equal-count halves but potentially unequal key ranges
- `range-partitioning-auto-split-manual-merge` — Splits are triggered automatically during `put()` when a partition exceeds `max_partition_size`, but merges require an explicit call to `merge_small_partitions()` and never happen implicitly
- `range-partitioning-merge-assumes-adjacency` — `Partition.merge(other)` appends other's keys without re-sorting; correctness depends on the caller passing the immediate right neighbor, with no runtime validation
- `range-partitioning-boundary-routing-bisect` — Key routing uses `bisect_right(boundaries, key) - 1` on a parallel sorted list of start keys, giving O(log p) partition lookup where p is the partition count

