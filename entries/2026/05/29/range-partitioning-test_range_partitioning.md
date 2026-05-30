# File: range-partitioning/test_range_partitioning.py

**Date:** 2026-05-29
**Time:** 13:58

I'll work with the test file content provided and what I can infer from it.

---

# `range-partitioning/test_range_partitioning.py`

## Purpose

This is the test suite for a **range-based partitioning** implementation — a core data distribution strategy from DDIA Chapter 6. Range partitioning divides a keyspace into contiguous, non-overlapping ranges, each owned by a single partition. Unlike hash partitioning, it preserves key ordering, enabling efficient range scans but risking hot spots on sequential writes.

The test file validates the full lifecycle of a `RangePartitionedStore`: insertion, lookup, deletion, automatic splitting when partitions grow too large, merging when they shrink, range scanning across partition boundaries, and the invariants that hold the partitioning scheme together.

## Key Components

The tests import two types from `range_partitioning`:

- **`RangePartitionedStore`** — The main store. Constructed with `max_partition_size` (split threshold) and optional `min_partition_size` (merge threshold). Exposes `put(key, value)`, `get(key)`, `delete(key)`, `range_scan(start, end?)`, `merge_small_partitions()`, `get_partition_info()`, `get_partition_for_key(key)`, plus properties `partition_count` and `total_keys`.

- **`Partition`** (used as info objects) — Returned by `get_partition_info()` and `get_partition_for_key()`. Has at least `start_key`, `end_key`, and `size` attributes. `start_key` is `""` for the first partition; `end_key` is `None` for the last (unbounded upper end).

### Test Functions

| Test | What it validates |
|------|-------------------|
| `test_basic_put_get_delete` | CRUD operations on a single partition. Confirms `get` returns `None` for missing keys, `delete` returns `True`/`False` for present/absent keys. |
| `test_auto_split_triggers` | Inserting 5 keys with `max_partition_size=4` forces a split to 2 partitions. |
| `test_split_produces_equal_partitions` | After splitting 11 keys, the two halves differ by at most 1 key — median-based splitting. |
| `test_boundaries_contiguous` | After many splits, partition boundaries form a seamless chain: first starts at `""`, last ends at `None`, each `end_key` equals the next `start_key`. |
| `test_range_scan_single_partition` | Range scan `[b, d)` returns keys `b, c` — confirming half-open interval semantics. |
| `test_range_scan_multiple_partitions` | Same semantics but the scan crosses partition boundaries transparently. |
| `test_range_scan_no_end` | Omitting `end_key` scans to the end of the keyspace. |
| `test_merge_small_partitions` | Deleting keys below `min_partition_size` triggers a merge back to 1 partition; data survives. |
| `test_merge_respects_threshold` | Merge is a no-op when combined size would exceed `min_partition_size`. |
| `test_routing_after_splits` | `get_partition_for_key` returns the correct partition metadata for each key post-split. |
| `test_large_scale_balanced` | 10,000 keys: no partition exceeds `max_partition_size`, boundaries are contiguous, total key count is preserved. |
| `test_delete_then_merge` | Bulk delete followed by merge consolidates partitions; surviving keys remain accessible. |
| `test_example_usage` | End-to-end walkthrough matching a spec/docstring example. |
| `test_update_value` | Overwriting a key updates the value without incrementing `total_keys`. |

## Patterns

**Progressive complexity testing.** Tests build from single-partition CRUD up to 10,000-key stress tests, each layer adding one concern (splitting, merging, scanning, routing).

**Invariant-based assertions.** Rather than testing implementation details, the suite checks structural invariants — contiguous boundaries, balanced sizes, correct routing — that must hold regardless of the internal splitting strategy.

**Half-open interval convention.** Range scans use `[start, end)` — inclusive start, exclusive end — matching the partition boundary model where `start_key` is inclusive and `end_key` is exclusive (or `None` for unbounded).

**Sentinel boundaries.** The first partition always starts at `""` (empty string, sorts before everything) and the last ends at `None` (unbounded above). This eliminates edge cases around the minimum and maximum keys.

## Dependencies

**Imports:** `pytest` (test framework), `range_partitioning.Partition` and `range_partitioning.RangePartitionedStore` (the system under test).

**Imported by:** Nothing imports this file. It's consumed by pytest. A companion `tester_test_range_partitioning.py` likely exists as a meta-test or test generator (visible in the repo tree).

## Flow

Each test follows the same pattern:

1. Construct a `RangePartitionedStore` with specific size thresholds.
2. Insert keys (triggering auto-splits when `max_partition_size` is exceeded).
3. Assert structural properties (partition count, boundary contiguity, key accessibility).
4. Optionally delete keys and call `merge_small_partitions()`.
5. Assert post-merge invariants.

The critical data flow insight: `put()` is not just a write — it's a write-then-maybe-split. The split is triggered internally when a partition exceeds `max_partition_size`, so tests must account for the partition count changing mid-insertion.

## Invariants

These are the invariants the test suite enforces:

1. **Boundary contiguity**: `info[i].end_key == info[i+1].start_key` for all adjacent partitions. No gaps, no overlaps.
2. **Boundary sentinels**: First partition starts at `""`, last partition ends at `None`.
3. **Size bound**: No partition exceeds `max_partition_size` (tested at 10k scale).
4. **Even splitting**: After a split, partition sizes differ by at most 1 (median split).
5. **Scan semantics**: `range_scan(start, end)` returns `[start, end)` — half-open, sorted by key.
6. **Scan without end**: `range_scan(start)` returns all keys `>= start`.
7. **Delete idempotency**: `delete` returns `True` on first call, `False` on subsequent calls for the same key.
8. **Get for missing keys**: Returns `None`, not an exception.
9. **Update semantics**: `put` on an existing key replaces the value without changing `total_keys`.
10. **Merge safety**: Merged partitions preserve all remaining data.

## Error Handling

The test suite reveals a **no-exception** API design. Missing keys return `None` from `get()`. Deleting a nonexistent key returns `False`. There are no tests for error paths like invalid key types or `None` keys, suggesting the implementation trusts callers to provide string keys — consistent with a teaching/reference implementation rather than a production system.

## Topics to Explore

- [file] `range-partitioning/range_partitioning.py` — The implementation: how splits find the median, how boundaries are maintained, how range scans merge results across partitions
- [function] `range-partitioning/range_partitioning.py:merge_small_partitions` — The merge threshold logic and how it decides which adjacent pairs to combine
- [file] `range-partitioning/tester_test_range_partitioning.py` — Meta-test or test generator that likely validates the test suite itself against a spec
- [general] `split-vs-hash-partitioning` — Compare this range-based approach with `consistent-hashing/` to understand the ordering-vs-distribution tradeoff from DDIA Chapter 6
- [function] `range-partitioning/range_partitioning.py:range_scan` — How the scan crosses partition boundaries — does it merge iterators or collect-and-sort?

## Beliefs

- `range-partition-scan-half-open` — `range_scan(start, end)` uses half-open interval semantics `[start, end)`, returning keys where `start <= key < end`
- `range-partition-split-median` — Auto-split divides a partition at the median key, producing two partitions whose sizes differ by at most 1
- `range-partition-boundaries-contiguous` — Partition boundaries form a seamless chain: first starts at `""`, last ends at `None`, and each partition's `end_key` equals the next partition's `start_key`
- `range-partition-no-exceptions-api` — The store uses return-value signaling (`None` for missing gets, `False` for redundant deletes) rather than raising exceptions
- `range-partition-merge-guards-threshold` — `merge_small_partitions` only merges adjacent partitions when their combined size stays below `min_partition_size`, preventing immediate re-splitting

