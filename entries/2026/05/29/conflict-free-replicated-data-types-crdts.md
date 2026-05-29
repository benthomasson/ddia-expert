# File: conflict-free-replicated-data-types/crdts.py

**Date:** 2026-05-29
**Time:** 09:03

# `conflict-free-replicated-data-types/crdts.py`

## Purpose

This file implements four state-based CRDTs (Conflict-Free Replicated Data Types) — the core data structures from DDIA Chapter 5 that allow multiple replicas to update independently and converge to the same state without coordination. It also provides `CRDTReplicaGroup`, a test harness that simulates a cluster of replicas with network sync.

Each CRDT satisfies the mathematical requirement for convergence: the `merge` operation is commutative, associative, and idempotent, meaning replicas can sync in any order, any number of times, and always reach the same result.

## Key Components

### `GCounter` — Grow-Only Counter

A counter that can only increase. Each replica maintains its own slot in a `{replica_id: count}` dictionary. `value()` sums all slots. Merge takes the element-wise `max` — this works because counts are monotonically increasing per replica, so `max` always preserves the highest observed value without double-counting.

### `PNCounter` — Positive-Negative Counter

Supports both increment and decrement by composing two `GCounter` instances: `p` for positive increments, `n` for negative (decrements). The value is `p.value() - n.value()`. This is a textbook example of building richer CRDTs by composing simpler ones — you can't subtract from a G-Counter, but you can track subtractions in a separate G-Counter.

### `LWWRegister` — Last-Writer-Wins Register

A single-value register where conflicts are resolved by timestamp, with `replica_id` as a deterministic tiebreaker (lexicographic comparison via Python's tuple ordering). Has an internal auto-incrementing `_clock` so callers don't need to supply timestamps manually. When merging, the `_clock` advances to `max(self._clock, other._clock)` to avoid future collisions.

### `ORSet` — Observed-Remove Set

The most complex CRDT here. Each `add()` creates a unique tag `(replica_id, seq_num)`. `remove()` tombstones all currently-known tags for an element. Merge unions active tags from both sides, then subtracts the union of both tombstone sets. This gives add-wins semantics: if replica A adds element X concurrently with replica B removing it, the add survives because its tag wasn't in B's tombstone set at removal time.

### `CRDTReplicaGroup` — Test Harness

Creates a set of replicas from any CRDT class and manages sync. `sync(from, to)` does a one-directional merge using `deepcopy` to simulate network serialization (the source is copied so mutation of the target doesn't affect the source). `sync_all()` runs two full rounds of all-pairs sync, which is sufficient for convergence in any state-based CRDT.

## Patterns

**Uniform CRDT interface.** Every CRDT follows the same shape: constructor takes `replica_id`, has a `merge(other)` that mutates and returns `self`, a `state()` for serialization, and `__eq__` for convergence checks. This lets `CRDTReplicaGroup` work polymorphically with any of them.

**Merge returns self.** All `merge` methods mutate in-place and return `self`. This is a fluent pattern that allows chaining but — more importantly — makes the merge direction unambiguous: the caller's state is updated, the argument is read-only.

**Composition over inheritance.** `PNCounter` doesn't subclass `GCounter`; it composes two instances. This mirrors the DDIA approach of building complex CRDTs from simple building blocks.

**Deep copy for network simulation.** `CRDTReplicaGroup.sync` uses `deepcopy` to prevent aliasing — without it, merging mutable sets/dicts would create shared references between replicas, hiding bugs where merge accidentally mutates the source.

## Dependencies

**Imports:** Only `deepcopy` from the standard library — no external dependencies. This keeps the CRDT implementations self-contained and focused on the algorithms.

**Imported by:** `test_crdts.py` and `tester_test_crdts.py` — the test suite and its meta-test validator.

## Flow

A typical usage cycle:

1. Create replicas (directly or via `CRDTReplicaGroup`)
2. Each replica applies local mutations (`increment`, `set`, `add`, `remove`)
3. Replicas exchange state via `merge` (or `sync`/`sync_all` through the group)
4. After sufficient sync rounds, all replicas converge to the same value

For `ORSet`, the internal flow during merge is:
1. Collect all elements referenced by either side
2. Union both tombstone sets
3. For each element, union active tags from both sides, subtract the merged tombstones
4. Keep only elements with surviving tags

## Invariants

- **GCounter monotonicity**: `increment` rejects negative amounts (`ValueError`). Per-replica counts only grow, guaranteeing the element-wise `max` in merge is a valid join operation.
- **LWW deterministic resolution**: Ties are broken by `(timestamp, writer_id)` tuple comparison — since replica IDs are unique, this is a total order. No two concurrent writes can result in ambiguity.
- **ORSet add-wins**: A concurrent add and remove of the same element results in the element being present, because the add's tag was created after the remove's tombstone snapshot.
- **ORSet tag uniqueness**: Tags are `(replica_id, seq_num)` with a per-replica monotonic counter. After merge, `_seq` advances to `max(self._seq, other._seq)` to prevent tag collisions.
- **Convergence guarantee**: `sync_all` runs 2 full rounds of all-pairs sync. For state-based CRDTs with commutative/associative/idempotent merge, this is sufficient for convergence regardless of prior state.

## Error Handling

Minimal by design — CRDTs have narrow failure modes:

- `GCounter.increment` raises `ValueError` on negative amounts — the only explicit validation.
- `ORSet.remove` returns `False` if the element isn't present rather than raising, which is the correct semantic (removing something you haven't seen is a no-op, not an error).
- No validation on merge inputs (e.g., type checking). The code trusts callers to merge compatible types, which is appropriate for an educational implementation.
- No handling of clock drift, Byzantine replicas, or serialization errors — those are outside the CRDT abstraction boundary.

## Topics to Explore

- [file] `conflict-free-replicated-data-types/test_crdts.py` — See how convergence, concurrent operations, and edge cases are tested for each CRDT type
- [general] `or-set-tombstone-growth` — The tombstone set in `ORSet` grows without bound; production systems (e.g., Riak) use causal context or garbage collection to address this
- [function] `conflict-free-replicated-data-types/crdts.py:LWWRegister.set` — The auto-clock mechanism and its interaction with explicit timestamps during merge is subtle and worth tracing through
- [file] `vector-clocks/vector_clock.py` — Vector clocks are the causal ordering primitive that more sophisticated CRDTs build on; compare with the simpler timestamp approach in `LWWRegister`
- [general] `delta-state-crdts` — These are state-based (CvRDTs); delta-state and operation-based (CmRDTs) variants trade different costs — worth understanding the tradeoffs discussed in DDIA

## Beliefs

- `crdt-merge-is-idempotent` — All four CRDT `merge` methods are idempotent: merging the same state twice produces the same result as merging once
- `pncounter-composes-gcounters` — `PNCounter` delegates entirely to two `GCounter` instances; it contains no independent merge or counting logic
- `orset-add-wins-semantics` — Concurrent add and remove of the same element in `ORSet` resolves in favor of the add, because the add's tag is not in the remover's tombstone set
- `lww-tiebreak-is-deterministic` — `LWWRegister.merge` uses `(timestamp, writer_id)` tuple comparison, making conflict resolution a total order with no ambiguity
- `sync-all-two-rounds-suffices` — `CRDTReplicaGroup.sync_all` runs exactly 2 rounds of all-pairs sync, which is sufficient for convergence of any state-based CRDT with a valid merge lattice

