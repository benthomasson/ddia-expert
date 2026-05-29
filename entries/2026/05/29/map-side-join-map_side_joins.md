# File: map-side-join/map_side_joins.py

**Date:** 2026-05-29
**Time:** 08:59

## Purpose

This file implements three **map-side join strategies** from DDIA Chapter 10 (Batch Processing). Map-side joins avoid the shuffle/reduce phase of a traditional MapReduce join by exploiting knowledge about the input data (size, partitioning, sort order). The file provides `BroadcastHashJoin`, `PartitionedHashJoin`, and `SortMergeJoin`, plus a `compare_join_strategies` function that runs all three and verifies they produce identical results.

Each strategy trades different assumptions about input data for different performance characteristics — the core pedagogical point of DDIA's treatment of joins.

## Key Components

### `JoinResult`
Simple container for join output: a list of `records` (dicts) and a `stats` dict with operational metrics (hash lookups, comparisons, records read, etc.). The `.count` property is sugar over `len(records)`.

### `BroadcastHashJoin`
**Assumption:** One dataset is small enough to fit in memory on every mapper.

- Constructor builds a hash table from the small dataset eagerly (`_build_hash_table`).
- `join()` simulates distributing the large dataset across `num_mappers` via round-robin chunking, then each "mapper" probes the hash table.
- Supports `"inner"` and `"left"` join types (left = large side preserved when unmatched).
- Tags each output record with `_mapper_id` to show which mapper produced it.

### `PartitionedHashJoin`
**Assumption:** Both datasets are partitioned on the join key using the same hash function, so matching records land in the same partition.

- `join()` partitions both datasets via `partition_dataset()`, then runs a local hash join within each partition.
- For left joins, tracks `matched_left_keys` per partition to emit unmatched left records afterward.
- Each partition acts as an independent mapper (`_mapper_id = part_id`).

### `SortMergeJoin`
**Assumption:** Both datasets are sorted by the join key (or can be sorted).

- `join()` checks sort order with `_is_sorted()`, sorts if needed, then does a linear merge.
- Equal keys produce a **cartesian product** within the matching group — it collects all left records with the same key, all right records with the same key, then emits every combination.
- For left joins, emits unmatched left records (those where `lk < rk`) with `None` fills via `_left_unmatched()`.
- Tracks whether sorting was needed in `stats["sorted_input"]`.

### Helper Functions

| Function | Contract |
|----------|----------|
| `_merge_records(left, right, left_key, right_key)` | Merges two dicts. The join key appears once. Conflicting field names get `left_`/`right_` prefixes. |
| `_build_hash_table(dataset, key)` | Returns `(defaultdict(list), skipped_count)`. Records missing the key are silently skipped. |
| `partition_dataset(dataset, key, n)` | Hash-partitions into `n` buckets using Python's `hash()`. Records missing the key are dropped. |
| `sort_dataset(dataset, key)` | Returns a new sorted list. Records missing the key are appended at the end unsorted. |
| `_is_sorted(dataset, key)` | Linear scan; skips records missing the key. |
| `compare_join_strategies(left, right, join_key)` | Runs all three strategies on inner join, normalizes results (strips `_mapper_id`, sorts), and returns a `verification` boolean confirming equivalence. |

## Patterns

**Simulated parallelism.** There are no threads or processes — mapper parallelism is modeled by chunking data and tagging output with `_mapper_id`. This keeps the code deterministic and testable while illustrating the concept.

**Conflict-prefixed merge.** When both sides share a field name (other than the join key), the merge disambiguates with `left_`/`right_` prefixes. This convention is applied consistently across all three join types, including the `None`-fill paths for left joins.

**Stats instrumentation.** Every join returns operational counters (hash lookups, comparisons, records read, skipped). This supports the pedagogical goal of comparing strategies quantitatively via `compare_join_strategies`.

**Defensive key-missing handling.** Every path that reads a join key checks for its presence first. Missing-key records are counted as `skipped_records` rather than raising exceptions.

## Dependencies

**Imports:** Only `collections.defaultdict` — no external dependencies.

**Imported by:** The test files `test_map_side_joins.py` and `tester_test_map_side_joins.py` exercise all three join strategies.

## Flow

Taking `BroadcastHashJoin` as the canonical example:

1. **Build phase** (constructor): Iterate small dataset → populate `defaultdict(list)` keyed on `small_key`.
2. **Distribute phase** (`join`): Round-robin large dataset records into `num_mappers` chunks.
3. **Probe phase**: For each large-side record, look up its key in the hash table.
   - Match → merge each matching small record with the large record.
   - No match + left join → emit the large record with `None` fills for small-side fields.
4. **Collect phase**: Accumulate all merged records, compute stats, return `JoinResult`.

`PartitionedHashJoin` replaces steps 1–2 with co-partitioning (both sides hashed into the same buckets), then runs the build/probe locally per partition.

`SortMergeJoin` replaces the hash table entirely with a two-pointer merge, grouping equal keys for cartesian product.

## Invariants

- **Join key appears exactly once** in merged output — `_merge_records` includes the left key and skips both keys from their respective records.
- **Field conflict resolution is symmetric** — `left_` prefix for left-side conflicts, `right_` for right-side, across all code paths including `None` fills.
- **`_mapper_id` is always set** on every output record, even for `SortMergeJoin` (hardcoded to `0`).
- **`compare_join_strategies` normalizes away `_mapper_id`** before comparing, so mapper assignment differences don't cause false verification failures.
- **Records missing the join key are never emitted** — they're counted in `skipped_records` and dropped.

## Error Handling

There is essentially none. The code takes a **silent-skip** approach: records missing the join key are filtered out and counted, but no exceptions are raised. `_build_hash_table` returns the skip count; `SortMergeJoin` accumulates it in a local variable. There's no validation on input types, no bounds checking on `num_partitions`/`num_mappers`, and no handling of unhashable key values (which would raise from `hash()` or dict lookup).

This is appropriate for a reference implementation — it prioritizes clarity of the algorithm over production robustness.

## Topics to Explore

- [file] `map-side-join/test_map_side_joins.py` — See which edge cases are tested (left joins, missing keys, field conflicts, multi-mapper verification)
- [function] `map_side_joins.py:compare_join_strategies` — The normalization logic reveals what the implementation considers semantically equivalent across strategies
- [file] `batch-word-count/pipeline.py` — Another Ch.10 batch processing implementation; compare how it models mapper parallelism
- [general] `ddia-ch10-reduce-side-joins` — How reduce-side joins (shuffle + sort + group) differ from these map-side approaches and when each is preferred
- [function] `map_side_joins.py:_merge_records` — The conflict-resolution logic is the trickiest shared code; understanding it clarifies all three join types

## Beliefs

- `map-side-join-three-strategies` — The module implements exactly three join strategies (broadcast hash, partitioned hash, sort-merge) that all produce identical inner-join results when verified by `compare_join_strategies`
- `map-side-join-conflict-prefix` — When left and right datasets share a non-key field name, the merged record uses `left_` and `right_` prefixes to disambiguate; this applies in all join types including None-fill paths
- `map-side-join-no-real-parallelism` — Mapper parallelism is simulated via round-robin chunking and `_mapper_id` tagging; no threads or processes are used
- `map-side-join-missing-key-skip` — Records missing the join key are silently dropped and counted in `stats["skipped_records"]` rather than raising exceptions
- `map-side-join-cartesian-on-duplicates` — When multiple records share the same join key value, all strategies produce the cartesian product of the matching groups

