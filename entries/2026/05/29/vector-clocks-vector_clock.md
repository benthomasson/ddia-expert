# File: vector-clocks/vector_clock.py

**Date:** 2026-05-29
**Time:** 09:04

# `vector-clocks/vector_clock.py`

## Purpose

This file implements **vector clocks** — the classic mechanism for tracking causal ordering in distributed systems (DDIA Chapter 5). It owns two responsibilities: (1) the vector clock data structure itself, which captures happens-before relationships between events across nodes, and (2) a `VersionedKVStore` that uses vector clocks to detect and manage write conflicts in a Dynamo-style key-value store.

## Key Components

### `VectorClock`

An **immutable** mapping of `{node_id: counter}`. Every mutation method returns a new instance rather than modifying in place.

- **`increment(node_id)`** — Returns a new clock with that node's counter bumped by 1. This is what a node calls when it performs a local event.
- **`merge(other)`** — Element-wise max across all keys from both clocks. Used when a node receives a message — the receiver merges its clock with the sender's to capture "I know about everything you knew about."
- **`compare(other)`** — The core partial-order comparison. Returns one of four strings: `BEFORE` (self happened-before other), `AFTER` (other happened-before self), `EQUAL`, or `CONCURRENT` (incomparable — neither dominates). This is determined by scanning all keys: if self has any component less *and* any component greater, the clocks are concurrent.
- **`dominates(other)`** — `self >= other` on every component. Equivalent to `compare` returning `AFTER` or `EQUAL`.
- **`prune(max_nodes)`** — Truncates the clock to keep only the `max_nodes` entries with the highest counters. This is a practical concern: in large clusters, vector clocks grow unboundedly; pruning trades precision for space, at the risk of introducing false conflicts.

The constructor strips zero-valued entries (`{k: v for ... if v > 0}`), so an empty dict and `{"A": 0}` produce identical clocks. This keeps equality and hashing consistent.

### `VersionedValue`

A simple `(value, vector_clock)` pair — a value tagged with the causal context in which it was written.

### `VersionedKVStore`

A per-node key-value store that maintains **sibling lists** — multiple concurrent versions of the same key.

- **`put(key, value, context=None)`** — Writes a value. Increments the node's clock, then merges it with the caller-supplied `context` (the clock from the version the client read before writing). It walks existing versions: any dominated by the new clock are dropped; any concurrent with it survive as siblings. Tracks whether a sibling was created via the `history` log.
- **`get(key)`** — Returns all current siblings for a key. A list with more than one element means there are unresolved conflicts.
- **`reconcile(key, merged_value, contexts)`** — The client-side conflict resolution path. Merges all sibling contexts together, increments the local clock, and replaces all siblings with a single resolved version.
- **`_receive_replica(key, value, vector_clock)`** — Anti-entropy ingest. Adds an incoming version as a sibling if it's concurrent with local versions, replaces local versions it dominates, and is silently dropped if dominated by any surviving local version.

### `find_conflicts(versions)`

A utility that checks whether any pair of versions in a list are concurrent — i.e., whether the client needs to reconcile.

## Patterns

**Immutable value objects.** `VectorClock` never mutates `_clock`; every operation returns a fresh instance. This avoids aliasing bugs that would be devastating in a distributed system where clocks are shared across messages and stored in multiple data structures.

**Dynamo-style sibling resolution.** The store doesn't pick a winner on conflict — it keeps all concurrent versions and pushes resolution to the caller via `reconcile()`. This matches Amazon Dynamo's design (and Riak's) where the application decides how to merge (e.g., shopping cart union).

**History tracking.** `_HistoryEntry` records every mutation with its action type (`write`, `sibling_created`, `reconciled`). This is purely for observability/testing — the store's correctness doesn't depend on it.

## Dependencies

**Imports:** Only stdlib — `dataclasses` and `typing`. No external dependencies.

**Imported by:** `test_vector_clock.py` and `tester_test_vector_clock.py` — the test suite and its meta-tests.

## Flow

A typical lifecycle:

1. **Client reads** `store.get("cart")` → gets `[VersionedValue(value="eggs", vc={A:1})]`
2. **Client writes** `store.put("cart", "eggs,milk", context=vc{A:1})` → store increments its clock to `{A:2}`, merges with context, drops the old version (dominated), stores new version.
3. **Concurrent write on node B** (no context from A's write) → `store_b.put("cart", "eggs,bread")` with `vc={B:1}`. When replicated to A, `_receive_replica` finds `{A:2}` and `{B:1}` are concurrent → both survive as siblings.
4. **Client reads** → gets two siblings. `find_conflicts()` returns `True`.
5. **Client reconciles** → `store.reconcile("cart", "eggs,milk,bread", [vc_a, vc_b])` → merges both clocks, increments, replaces siblings with one resolved value.

## Invariants

- **Zero-stripping**: The clock never contains entries with value 0. This is enforced in `__init__`, which means two clocks built from different dicts but with the same non-zero entries are `==` and hash identically.
- **Monotonic clocks**: `increment` and `merge` can only increase component values. There's no decrement operation. The store's `_clock` is merged with every outgoing and incoming version clock, so it's always at least as large as any version it has seen.
- **Sibling preservation**: `put` never drops a version it can't prove is dominated. If the new write and an existing version are concurrent, both survive. This ensures no writes are silently lost.
- **Reconciliation collapses all siblings**: `reconcile` unconditionally replaces `self._data[key]` with a single-element list. It doesn't check whether the contexts passed actually cover all existing siblings — the caller is trusted to have read them all.

## Error Handling

Essentially none. The code trusts its callers:
- `get()` on a missing key returns `[]` (not an error).
- `put()` with no context creates a fresh version — this is the "first write" path, not an error case.
- `prune()` with `max_nodes >= len(clock)` is a no-op copy.
- No validation that `node_id` is non-empty, that `value` is a string, or that `contexts` in `reconcile` are non-empty. This is appropriate for a reference implementation where the focus is on the algorithm, not production hardening.

## Topics to Explore

- [file] `vector-clocks/test_vector_clock.py` — See the test scenarios for multi-node conflict creation and reconciliation to understand the sibling lifecycle end-to-end
- [file] `conflict-free-replicated-data-types/crdts.py` — CRDTs solve the same conflict problem but with automatic merge semantics rather than sibling-based resolution; compare the tradeoffs
- [file] `lamport-clocks/lamport.py` — Lamport clocks are the simpler predecessor to vector clocks: they give total ordering but can't detect concurrency. Compare what `compare()` can express in each
- [general] `dynamo-anti-entropy` — The `_receive_replica` method implements one side of anti-entropy; explore how gossip protocols (see `gossip-protocol/`) would drive the replication that feeds it
- [function] `vector-clocks/vector_clock.py:prune` — Pruning trades correctness for space; explore under what conditions false conflicts are introduced and how Riak/Dynamo handle this in practice

## Beliefs

- `vector-clock-immutability` — `VectorClock` is immutable: `increment`, `merge`, and `prune` all return new instances and never modify `_clock` in place
- `sibling-preservation-on-concurrent-writes` — `VersionedKVStore.put` never drops an existing version unless the new version's clock dominates it; concurrent versions are preserved as siblings
- `reconcile-replaces-all-siblings` — `reconcile` unconditionally replaces all siblings for a key with a single merged version, regardless of whether the provided contexts cover every existing sibling
- `zero-entries-stripped` — `VectorClock.__init__` strips all entries with value 0, ensuring two clocks with identical non-zero entries are always equal and hash-equal
- `compare-returns-four-states` — `VectorClock.compare` returns exactly one of `BEFORE`, `AFTER`, `EQUAL`, or `CONCURRENT`, implementing the partial order defined by component-wise comparison

