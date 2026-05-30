# File: vector-clocks/test_vector_clock.py

**Date:** 2026-05-29
**Time:** 14:06

## Purpose

`test_vector_clock.py` is the test suite for a vector clock implementation — a core causality-tracking mechanism from distributed systems (DDIA Chapter 5). It validates four exported components: `VectorClock` (the logical clock itself), `VersionedValue` (a value tagged with its causal history), `VersionedKVStore` (a Dynamo-style versioned key-value store), and `find_conflicts` (a predicate that detects concurrent siblings).

The file owns **behavioral specification** for the entire module. It covers the full lifecycle: clock arithmetic → store operations → conflict creation → conflict detection → reconciliation.

## Key Components

The test suite is organized into 10 numbered test classes, each targeting a specific concern:

| Class | Tests | What it validates |
|-------|-------|-------------------|
| `TestComparison` | 5 | Partial ordering: BEFORE, AFTER, EQUAL, CONCURRENT |
| `TestIncrement` | 3 | Immutable tick of a single node's counter |
| `TestMerge` | 3 | Pointwise-max merge of two clocks |
| `TestDominates` | 4 | `dominates` (≥ on all entries) and `descends_from` |
| `TestStorePutGet` | 3 | Basic KV operations, including missing-key behavior |
| `TestConflictCreation` | 3 | How concurrent writes produce sibling versions |
| `TestConflictDetection` | 4 | `find_conflicts` over lists of `VersionedValue` |
| `TestReconciliation` | 2 | Client-side merge that collapses siblings into one |
| `TestReadModifyWrite` | 2 | Causal chains where each write carries forward context |
| `TestEdgeCases` | 7 | Empty clocks, single-node, many-writer fan-out, pruning, immutability |

## Patterns

**Immutability-by-test.** Several tests assert that operations like `increment` return a *new* `VectorClock` without mutating the original (`test_immutability`, `test_increment_existing`). This is the functional/persistent data structure pattern — critical for vector clocks because multiple code paths may hold references to the same clock.

**Multi-store simulation.** Concurrent writes are modeled by creating separate `VersionedKVStore` instances with different node IDs (`"A"`, `"B"`), writing independently, then merging via `_receive_replica`. This mirrors Dynamo's anti-entropy: each store is a replica coordinator, and conflict is surfaced when replicas exchange versions.

**Context-passing for causal chains.** The `context` parameter on `put` threads a vector clock forward, so the store knows the write *descends from* a prior version. Without context, writes are treated as fresh (concurrent with everything). This is the read-modify-write pattern from Dynamo/Riak.

**Sibling semantics.** When `get` returns multiple `VersionedValue` entries, the store has detected concurrent versions. The client is responsible for calling `reconcile` to merge them — the store never auto-resolves.

## Dependencies

**Imports:** `pytest` (test framework) and four symbols from `vector_clock.py`:
- `VectorClock` — the clock data structure
- `VersionedValue` — a `(value, vector_clock)` pair
- `VersionedKVStore` — the versioned store with `put`, `get`, `reconcile`, `_receive_replica`
- `find_conflicts` — standalone predicate

**Imported by:** `tester_test_vector_clock.py` likely wraps or validates this test suite (following the repo's `tester_test_*.py` convention).

## Flow

The tests exercise three distinct flows:

1. **Clock arithmetic** (classes 1–4): Construct `VectorClock` instances from dicts, call `compare`/`increment`/`merge`/`dominates`, assert return values. Pure functions, no state.

2. **Store operations** (classes 5–6, 9): Create a `VersionedKVStore` with a node ID → `put` returns the new vector clock → `get` returns a list of `VersionedValue` → assert sibling count and values. The `context` parameter on `put` is how the test communicates causal history.

3. **Conflict resolution** (classes 7–8): Build `VersionedValue` lists manually or via store operations → call `find_conflicts` or `store.reconcile` → assert the conflict is detected or resolved. `reconcile` takes a merged value and the list of sibling clocks, produces a new clock that dominates all siblings.

## Invariants

- **Partial order correctness:** If `vc1.compare(vc2) == "BEFORE"`, then `vc2.compare(vc1) == "AFTER"`. Tested symmetrically in `TestComparison`.
- **Merge supremacy:** `merge(a, b)` must dominate both `a` and `b` (pointwise max). Tested indirectly via equality checks.
- **Causal writes suppress siblings:** A `put` with `context=prior_clock` must replace (not coexist with) the version that clock represents. Tested in `test_sequential_writes_no_siblings` and `test_rmw_chain`.
- **Concurrent writes produce siblings:** Two writes without a causal relationship must both survive in `get`. Tested in `test_concurrent_writes_create_siblings`.
- **Reconciliation dominance:** After `reconcile`, the resulting clock `dominates` every sibling clock. Explicitly asserted in `test_reconcile_siblings`.
- **Immutability:** `increment` on a `VectorClock` must not mutate the original instance.
- **Prune keeps top-k:** `prune(k)` retains the `k` highest-count entries, zeroing the rest.

## Error Handling

The test suite doesn't test error paths — no `pytest.raises` blocks, no invalid input tests. The implicit contract is that `VectorClock` accepts any `dict[str, int]`, `get` returns `[]` for missing keys (not an exception), and operations on empty clocks are well-defined (tested in `TestEdgeCases`).

## Topics to Explore

- [file] `vector-clocks/vector_clock.py` — The implementation behind all four exported types; understanding `_receive_replica` internals is key to the conflict model
- [function] `vector-clocks/vector_clock.py:VectorClock.compare` — The partial-order implementation that drives BEFORE/AFTER/CONCURRENT classification
- [function] `vector-clocks/vector_clock.py:VersionedKVStore.reconcile` — How sibling clocks are merged and how the store prunes dominated versions post-reconciliation
- [general] `dynamo-style-versioning` — The Amazon Dynamo paper's approach to conflict detection and client-side resolution that this module implements
- [file] `vector-clocks/tester_test_vector_clock.py` — The meta-test layer that validates this test suite itself

## Beliefs

- `vector-clock-immutable-operations` — `VectorClock.increment` returns a new instance; the original is never mutated, enabling safe sharing across concurrent code paths
- `versioned-store-sibling-semantics` — `VersionedKVStore.get` returns multiple `VersionedValue` entries when concurrent (causally unrelated) writes exist; the store never auto-resolves conflicts
- `reconcile-dominates-all-siblings` — After `VersionedKVStore.reconcile`, the returned vector clock dominates every input sibling clock, guaranteeing the merged version causally supersedes all predecessors
- `context-parameter-establishes-causality` — Passing a vector clock as `context` to `put` declares the write descends from that version; without it, the write is treated as concurrent with existing versions
- `find-conflicts-detects-concurrency` — `find_conflicts` returns `True` if and only if at least two `VersionedValue` entries in the list have concurrent (non-dominating) vector clocks

