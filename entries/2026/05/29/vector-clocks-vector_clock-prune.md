# Function: prune in vector-clocks/vector_clock.py

**Date:** 2026-05-29
**Time:** 14:09



## `VectorClock.prune(max_nodes)` — Bounded Vector Clock Truncation

### Purpose

`prune` caps the size of a vector clock by discarding entries with the lowest counters, keeping only the `max_nodes` most-active nodes. This exists because vector clocks grow unboundedly as new nodes participate in the system — every unique node ID that ever increments the clock adds an entry. In long-running systems with node churn (nodes joining and leaving), the clock can grow without limit. Pruning trades causal accuracy for bounded space: you lose the ability to detect causality relationships involving the dropped nodes, but the clock stays a fixed size.

This is the strategy described in DDIA and used by systems like Riak — when the clock gets too large, drop the least-active entries on the assumption that nodes with low counters are the least relevant to ongoing causal ordering.

### Contract

- **Precondition**: `max_nodes` should be a positive integer. The code doesn't enforce this — passing 0 or negative values would return an empty clock, and passing a non-integer would raise a `TypeError` from `sorted`/slicing.
- **Postcondition**: The returned clock has at most `max_nodes` entries, all with positive counters (guaranteed by the constructor's zero-stripping). The entries retained are those with the highest counter values.
- **Invariant**: The original clock is never mutated. A new `VectorClock` is always returned.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `max_nodes` | `int` | Upper bound on the number of node entries to retain. If the clock already has this many or fewer entries, it's returned as-is (copied). |

**Edge cases**: If `max_nodes >= len(self._clock)`, no pruning occurs. If multiple nodes share the same counter value, which ones survive the cut is determined by Python's `sorted` stability (preserving original dict iteration order among ties), but this is not a guarantee the caller should rely on.

### Return Value

A new `VectorClock` instance. The caller gets back a clock that may have lost causal information — any subsequent `compare()` or `dominates()` call against another clock that references pruned nodes will behave as though those nodes were never seen (counter = 0 via `get()`'s default). This can cause false concurrency or false dominance.

### Algorithm

1. **Short-circuit**: If the clock already fits within `max_nodes`, copy the internal dict into a new `VectorClock` and return. This avoids the sort.
2. **Sort**: Convert `_clock.items()` to a list sorted by counter value descending. Nodes that have been incremented the most sort first.
3. **Truncate**: Slice the first `max_nodes` entries and construct a new `VectorClock` from them. The remaining entries are silently dropped.

### Side Effects

None. The method is pure — no mutation of `self`, no I/O, no state changes. Consistent with the class's immutable design (every mutating operation returns a new instance).

### Error Handling

No explicit error handling. Relies on Python built-ins raising naturally:
- `TypeError` if `max_nodes` isn't comparable to `int` in the slice operation
- No `ValueError` checks for negative values

### Usage Patterns

Pruning is typically applied periodically or at replication boundaries — e.g., before sending a vector clock over the wire, or when a store detects the clock has grown past a threshold. A typical call site might look like:

```python
vc = vc.increment(node_id)
if len(vc._clock) > MAX_VC_SIZE:
    vc = vc.prune(MAX_VC_SIZE)
```

The caller must accept that pruning is lossy. After pruning, causal relationships involving dropped nodes become undetectable, which can lead to false conflict detection (concurrent when it should be ordered) or, worse, silent overwrites (ordered when it should be concurrent).

### Dependencies

Only the `VectorClock` class itself (constructor and `sorted` built-in). No external modules.

---

## Topics to Explore

- [function] `vector-clocks/vector_clock.py:compare` — Understanding what breaks when pruned entries cause `get()` to return 0 for nodes that were actually active
- [function] `vector-clocks/vector_clock.py:merge` — How merge interacts with pruned clocks — merging before vs. after pruning yields different results
- [file] `vector-clocks/test_vector_clock.py` — Check whether tests cover the lossy semantics of pruning (false concurrency after prune)
- [general] `vector-clock-pruning-strategies` — Alternative pruning strategies: timestamp-based eviction (drop oldest by wall-clock), or the "crdt-style" approach of folding pruned entries into a summary counter
- [general] `dynamo-style-conflict-resolution` — How Riak/Dynamo handle the practical consequences of vector clock pruning in production (sibling explosion)

---

## Beliefs

- `vc-prune-keeps-highest-counters` — `prune(n)` retains the `n` entries with the highest counter values, discarding the rest
- `vc-prune-is-lossy` — Pruning permanently discards causal information; subsequent `compare()` calls may return incorrect results for interactions involving dropped nodes
- `vc-prune-no-mutation` — `prune` follows the class's immutable pattern: it returns a new `VectorClock` and never modifies `self._clock`
- `vc-prune-tiebreak-unstable` — When multiple nodes share the same counter value, which survive pruning depends on dict iteration order via `sorted` stability, not on any deterministic tiebreak policy

