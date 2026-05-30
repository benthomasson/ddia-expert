# File: map-side-join/test_map_side_joins.py

**Date:** 2026-05-29
**Time:** 13:28

I can't read the implementation file due to permissions, but the test file itself is very revealing. Here's the explanation based on what's visible.

---

# `map-side-join/test_map_side_joins.py`

## Purpose

This is the test suite for a module that implements three **map-side join strategies** from DDIA Chapter 10 (batch processing). Map-side joins avoid shuffling data across the network by exploiting known properties of the input data (pre-sorted, pre-partitioned, or small enough to broadcast). The test file validates that all three strategies — `BroadcastHashJoin`, `PartitionedHashJoin`, and `SortMergeJoin` — produce correct and consistent results across a variety of join scenarios.

## Key Components

### Fixtures

- **`users`** — A 4-record "small" dataset keyed by `user_id`. Represents the dimension/lookup table.
- **`orders`** — A 5-record "large" dataset keyed by `user_id`. Includes `user_id: 5` which has no match in `users`, and `user_id: 1` which has two orders — these are deliberate edge cases for testing unmatched records and one-to-many fan-out.

### Strategies Under Test

| Class | Pattern | Key Characteristic |
|---|---|---|
| `BroadcastHashJoin` | Small dataset broadcast to all mappers | Accepts `num_mappers` param; builds hash table from small side |
| `PartitionedHashJoin` | Both sides partitioned by join key | Accepts `num_partitions`; each partition joined independently |
| `SortMergeJoin` | Both sides sorted, then merged | Detects whether input is pre-sorted (`stats["sorted_input"]`) |

All three return a `JoinResult` object with `.count`, `.records`, and `.stats` attributes.

### Utility Functions

- **`partition_dataset(dataset, key, num_partitions)`** — Distributes records across N partitions by join key.
- **`sort_dataset`** — Imported but not directly tested (used internally by `SortMergeJoin`).
- **`compare_join_strategies`** — Runs all three strategies and cross-checks that results agree, returning a dict with per-strategy results and a `verification` flag.

## Patterns

**Strategy equivalence testing.** The central design pattern is running the same logical join through all three strategies and asserting identical output. `test_inner_join_all_strategies` does this explicitly; `test_large_dataset` does it at scale (1000 × 2000 records). `compare_join_strategies` wraps this into a utility function.

**Semantic edge cases as first-class tests.** The suite systematically hits: inner vs. left joins, one-to-many, many-to-many (Cartesian product), no matches, empty datasets on both sides, missing join keys, field name collisions, sorted vs. unsorted input, and varying mapper/partition counts. Each is a separate test rather than parameterized — this keeps failure messages unambiguous.

**API asymmetry across strategies.** The three strategies have deliberately different APIs reflecting their different assumptions:
- `BroadcastHashJoin`: small side provided at construction, large side at `.join()` time
- `PartitionedHashJoin`: both sides provided at `.join()` time
- `SortMergeJoin`: both sides provided at `.join()` time

This mirrors real-world constraints: broadcast joins need the small dataset loaded upfront.

## Dependencies

**Imports:** `pytest` for test infrastructure, everything else from `map_side_joins` (the sibling implementation module).

**Imported by:** Nothing imports this. It's a leaf test file. The sibling `tester_test_map_side_joins.py` likely runs or wraps these tests for the project's meta-testing harness.

## Flow

Most tests follow the same pattern:

1. Construct a join strategy with configuration (mapper count, partition count, key name)
2. Call `.join(dataset(s), join_type="inner"|"left")`
3. Assert on `result.count` for cardinality
4. Assert on `result.records` for specific content (order IDs, user IDs, field values)
5. Assert on `result.stats` for operational metadata (records read, skipped, mapper IDs)

The `test_large_dataset` test seeds `random` with 42 for determinism — all 2000 right-side keys fall in `[0, 999]`, guaranteeing every right record matches, so the expected count is exactly 2000.

## Invariants

- **Inner join produces only matched records.** Every inner join test asserts that `order_id: 105` (user_id 5, no matching user) is excluded.
- **Left join preserves all left-side records.** The broadcast left join keeps the unmatched order (user_id 5). The partitioned and sort-merge left joins keep the unmatched user (user_id 4). This distinction comes from which dataset is "left" — for broadcast, the large/streamed side is left.
- **Many-to-many produces a Cartesian product.** 2 left × 3 right = 6 output records.
- **Strategy equivalence.** All three strategies produce the same count for the same input and join type.
- **Missing keys are skipped, not errored.** Records without the join key are silently skipped and counted in `stats["skipped_records"]`.
- **Field name conflicts are resolved with prefixing.** When both sides have a `value` field, the output contains `left_value` and `right_value`.
- **Mapper count doesn't affect correctness.** `test_broadcast_mapper_counts` proves this for 1, 2, 4, and 8 mappers.

## Error Handling

The test suite reveals that the implementation uses a **tolerant** error strategy:
- Missing join keys → record silently skipped, tracked in `stats["skipped_records"]` (tested in `test_missing_join_key`)
- Empty datasets → valid empty `JoinResult` with `count == 0` (tested in `test_empty_datasets`)
- No exceptions are expected or tested for. The API surface is designed to never raise on valid join types with well-formed (dict-of-dicts) input.

---

## Topics to Explore

- [file] `map-side-join/map_side_joins.py` — The implementation of all three join strategies, including hash table construction, partition assignment, merge logic, and the `JoinResult` dataclass
- [function] `map-side-join/map_side_joins.py:BroadcastHashJoin` — How the small dataset is replicated to each mapper and how the hash table is built and probed during the join
- [function] `map-side-join/map_side_joins.py:SortMergeJoin` — The sorted-input detection logic and the merge algorithm, especially how it handles duplicates for many-to-many joins
- [general] `map-side-join-vs-reduce-side-join` — DDIA Ch. 10 contrasts map-side and reduce-side joins; understanding when each is appropriate (pre-partitioned/sorted data vs. arbitrary input)
- [file] `map-side-join/tester_test_map_side_joins.py` — The meta-test wrapper that likely validates the test suite itself against expected behaviors

## Beliefs

- `broadcast-join-loads-small-side-at-construction` — `BroadcastHashJoin` receives the small dataset at construction time and builds a hash table; the large dataset is streamed through `.join()`, reflecting the real-world pattern where the small side is broadcast to all mapper nodes
- `all-three-join-strategies-produce-identical-inner-join-results` — For the same input data, `BroadcastHashJoin`, `PartitionedHashJoin`, and `SortMergeJoin` all produce the same set of output records on inner joins, verified at both small (4×5) and large (1000×2000) scale
- `missing-join-keys-are-skipped-not-errored` — Records lacking the join key field are silently excluded from results and counted in `stats["skipped_records"]` rather than raising an exception
- `field-name-conflicts-resolved-with-left-right-prefix` — When both sides of a join contain the same field name, the output record prefixes the conflicting fields with `left_` and `right_` to avoid data loss
- `sort-merge-join-detects-presorted-input` — `SortMergeJoin` inspects whether input is already sorted and reports this via `stats["sorted_input"]`, which allows it to skip the sort step on pre-sorted data

