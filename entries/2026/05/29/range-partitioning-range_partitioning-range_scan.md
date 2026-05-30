# Function: range_scan in range-partitioning/range_partitioning.py

**Date:** 2026-05-29
**Time:** 13:59

## `Partition.range_scan`

### Purpose

`range_scan` performs a sorted range query over a partition's in-memory key-value store. It returns all entries where `start <= key < end` (half-open interval), which is the standard convention in range-partitioned systems. This exists because range queries are the primary advantage of range partitioning over hash partitioning — co-locating lexicographically adjacent keys makes scans efficient.

### Contract

**Preconditions:**
- `self._keys` is sorted in ascending lexicographic order (maintained by `put`, which uses `bisect_left` for insertion).
- `start` is a string. No enforcement that `start < end` — if `start >= end`, the result is simply empty.

**Postconditions:**
- Returns pairs in sorted key order.
- The returned list is a new object (via `list(zip(...))`), so callers can mutate it freely.
- No mutation of internal state.

**Invariant:** The result set is exactly `{(k, v) | k in self._keys, start <= k < end}`, ordered by key.

### Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `start` | `str` | Inclusive lower bound of the scan range |
| `end` | `Optional[str]` | Exclusive upper bound. `None` means "scan to the end of the partition" |

**Edge cases:**
- `start == end` → empty result (bisect_left returns the same index for both).
- `start > end` → empty result (left > right after bisect, slice is empty).
- `end is None` → scans from `start` through the last key in the partition.
- `start` beyond all keys → empty result.
- Empty partition (`_keys == []`) → empty result regardless of arguments.

### Return Value

`list[tuple[str, Any]]` — a list of `(key, value)` pairs in ascending key order. Always a fresh list; can be empty. The caller does not need to handle errors — this method cannot raise under normal conditions.

### Algorithm

1. **Find left boundary:** `bisect.bisect_left(self._keys, start)` locates the index of the first key `>= start`. This is the inclusive start of the result window.

2. **Find right boundary:**
   - If `end is None`, set `right = len(self._keys)` — include everything through the end.
   - Otherwise, `bisect.bisect_left(self._keys, end)` locates the first key `>= end`. Since the interval is exclusive on the right, this is exactly where to stop — any key equal to `end` is excluded because `bisect_left` returns its position, and slicing with `[:right]` excludes index `right`.

3. **Slice and zip:** `self._keys[left:right]` and `self._values[left:right]` extract the matching keys and values as parallel slices, then `zip` pairs them up and `list()` materializes the result.

The algorithm is O(log n) for the two binary searches plus O(k) for materializing `k` result pairs.

### Side Effects

None. This is a pure read operation — no mutation of `_keys`, `_values`, or any other state.

### Error Handling

No explicit error handling. The method relies on `bisect_left` and Python slice semantics being safe for all inputs (out-of-range slices return empty lists, not exceptions). There are no raised exceptions.

### Usage Patterns

This method is called in two contexts:

1. **Directly on a partition** — when you know which partition to scan.
2. **From `RangePartitionedStore.range_scan`** (line ~128) — the store identifies all partitions whose key ranges overlap `[start_key, end_key)`, then calls `partition.range_scan(start_key, end_key)` on each and concatenates results. Note that the store passes the *global* `start_key`/`end_key` to every partition, not the partition's own boundaries — this works because `bisect_left` naturally clamps: keys outside the partition simply aren't found, so the slice is empty or partial as appropriate.

**Caller obligation:** The caller must ensure `start` and `end` are comparable strings. Since `_keys` is `list[str]`, passing a non-string would cause a `TypeError` from the comparison operators inside `bisect_left`.

### Dependencies

- **`bisect.bisect_left`** — standard library binary search. Returns the leftmost insertion point for a value in a sorted list. This is the only external dependency.
- **Internal invariant on `_keys`/`_values`** — these two lists must be parallel (same length, same indices correspond) and `_keys` must be sorted. This is maintained by `put`, `delete`, `split`, and `merge`.

### Assumptions Not Enforced by Types

1. **`_keys` is sorted.** Nothing in the type system guarantees this. If someone appends directly to `_keys` bypassing `put`, `range_scan` returns incorrect results silently.
2. **`_keys` and `_values` are parallel.** If their lengths diverge, `zip` silently truncates to the shorter list — you lose data without any error.
3. **`start < end` when `end` is not `None`.** No validation. Reversed ranges silently return empty results rather than raising.
4. **String comparisons are sufficient for ordering.** The code uses Python's default lexicographic string comparison. If keys represent numbers (e.g., `"9"` vs `"10"`), the ordering is lexicographic, not numeric — `"9" > "10"` is `True`.

## Topics to Explore

- [function] `range-partitioning/range_partitioning.py:RangePartitionedStore.range_scan` — The store-level scan that fans out across partitions and handles boundary alignment; shows how partition-level scans compose
- [function] `range-partitioning/range_partitioning.py:Partition.put` — The insertion method that maintains the sorted invariant that `range_scan` depends on
- [function] `range-partitioning/range_partitioning.py:Partition.split` — Splitting a partition at the median key; directly affects which partitions a range scan must touch
- [file] `range-partitioning/test_range_partitioning.py` — Test cases that exercise cross-partition scans, edge cases with `None` end keys, and split/merge interactions
- [general] `bisect-module-semantics` — Understanding the difference between `bisect_left` and `bisect_right` and why `bisect_left` is correct for both bounds here

## Beliefs

- `range-scan-half-open-interval` — `Partition.range_scan` implements a half-open interval `[start, end)`: start is inclusive, end is exclusive, matching the partition boundary convention `[start_key, end_key)`
- `range-scan-no-mutation` — `Partition.range_scan` is a pure read with no side effects; it never modifies `_keys` or `_values`
- `range-scan-relies-on-sorted-keys` — `range_scan` correctness depends on `_keys` being sorted, an invariant maintained by `put`/`delete`/`split`/`merge` but not enforced structurally
- `range-scan-none-end-scans-unbounded` — Passing `end=None` to `range_scan` scans from `start` through the last key in the partition with no upper bound
- `store-range-scan-passes-global-bounds` — `RangePartitionedStore.range_scan` passes the same global `start_key`/`end_key` to every relevant partition rather than clamping to partition boundaries, relying on `bisect_left` to naturally exclude out-of-range keys

