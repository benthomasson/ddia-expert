# File: multi-leader-replication/multi_leader.py

**Date:** 2026-05-29
**Time:** 07:01

# `multi-leader-replication/multi_leader.py`

## Purpose

This file implements **multi-leader (multi-master) replication** as described in Chapter 5 of *Designing Data-Intensive Applications*. It models a cluster where multiple nodes can independently accept writes, then asynchronously replicate changes to each other — the defining characteristic that separates multi-leader from single-leader replication. The file owns conflict detection, conflict resolution, and replication topology.

## Key Components

### `_TOMBSTONE`

A sentinel object used as the stored value when a key is deleted. Using `object()` guarantees identity-uniqueness — no real value can ever `is`-match it. This is a classic soft-delete pattern: deletes are writes, not erasures, so they can replicate and resolve conflicts like any other mutation.

### `ConflictStrategy` (Enum)

Two resolution policies:
- **`LAST_WRITE_WINS`** — deterministic tiebreak on `(timestamp, node_id)` tuple comparison. Simple, but silently drops the losing write.
- **`CUSTOM_MERGE`** — delegates to a caller-supplied `merge_fn(key, local_val, remote_val, local_ts, remote_ts)`. This is DDIA's "application-level" resolution — the only strategy that can preserve both sides of a conflict.

### `Topology` (Enum)

Controls how changes fan out during `sync()`:
- **`ALL_TO_ALL`** — every node sends to every other node. Convergence in one round.
- **`RING`** — each node sends only to its successor. Requires N-1 rounds to fully propagate through an N-node cluster.

### `ConflictRecord` (dataclass)

An audit trail entry capturing both sides of a conflict and how it was resolved. The `resolved_value` and `resolved_by` fields make conflicts inspectable after the fact — critical for debugging in multi-leader systems where silent data loss is the main risk.

### `ReplicaNode`

The core abstraction. Each node maintains:

| Field | Purpose |
|-------|---------|
| `_clock` | Lamport scalar clock, incremented on every local write and bumped on every remote apply |
| `_store` | `dict[str, (value, timestamp, origin_node_id, is_tombstone)]` — the data, with full provenance |
| `_pending` | Outgoing replication log — changes queued for the next `sync()` |
| `_conflict_log` | Append-only list of `ConflictRecord`s |
| `_seen` | Per-key set of `(timestamp, origin)` pairs for idempotency |

**Contract**: After `put`/`delete`, the change is in `_store` *and* in `_pending`. After `get_pending_changes()`, the pending list is drained (returned and cleared). This is a consume-once queue.

### `MultiLeaderCluster`

Orchestrator that wires nodes together. Provides the replication transport (`sync()`) and convergence checking (`all_converged()`, `sync_until_converged()`). It separates the *mechanism* of conflict resolution (in `ReplicaNode`) from the *policy* of when and how to replicate (in `MultiLeaderCluster`).

## Patterns

**Lamport clock for causal ordering.** `_tick()` increments before every local write. `apply_remote_change` merges with `max(local, remote) + 1`. This ensures the clock always advances and establishes a total order over events when combined with node ID as a tiebreaker.

**Tombstone-based deletion.** Deletes write a `_TOMBSTONE` value with `is_tombstone=True` rather than removing the key. This is essential in multi-leader systems — without tombstones, a delete on node A would be indistinguishable from "never existed" when node B tries to replicate, and the value would silently reappear.

**Idempotent apply.** The `_seen` set prevents the same change from being applied twice. This matters especially in ring topology, where a change propagates through multiple hops and could revisit a node.

**Deterministic tiebreaking.** LWW compares `(timestamp, node_id)` tuples. Since Python tuple comparison is lexicographic, equal timestamps break ties by node ID string comparison. This guarantees all nodes reach the same resolution independently — no coordination needed.

**Separate collection and distribution.** `sync()` collects all pending changes *first*, then distributes. This prevents a change from cascading within a single sync round (node A's change reaching B, then B's copy reaching C in the same round).

## Dependencies

**Imports**: Only stdlib — `enum.Enum`, `typing`, `dataclasses.dataclass`. No external dependencies. This is a pure in-memory simulation.

**Imported by**: `test_multi_leader.py` and `tester_test_multi_leader.py` — test suites that exercise the replication and conflict resolution logic.

## Flow

### Write path
1. Client calls `node.put(key, value)` or `node.delete(key)`
2. Lamport clock ticks → timestamp assigned
3. Entry written to `_store` with `(value, ts, self.node_id, is_tombstone)`
4. Change dict appended to `_pending`
5. `(ts, node_id)` recorded in `_seen`

### Replication path
1. `cluster.sync()` drains `_pending` from every node
2. Based on topology, each change is delivered to target nodes via `apply_remote_change`
3. `apply_remote_change`:
   - Checks `_seen` for idempotency → skip if duplicate
   - Merges Lamport clock: `max(local, remote) + 1`
   - If no local entry exists → accept unconditionally
   - If same origin + same timestamp → skip (already have it)
   - If different origin → **conflict**:
     - LWW: higher `(ts, node_id)` wins
     - CUSTOM_MERGE: call `merge_fn`, assign new timestamp `max(local_ts, remote_ts) + 1`
   - Appends change to `_pending` for further propagation (ring topology needs this)
   - Returns `ConflictRecord` if conflict occurred

### Convergence path
`sync_until_converged()` loops `sync()` up to `max_rounds` times, checking `all_converged()` after each round. Convergence means all nodes agree on the same `(value, timestamp, is_tombstone)` for every key.

## Invariants

1. **Lamport clock monotonicity.** `_clock` never decreases. `_tick()` increments by 1; `apply_remote_change` sets `max(local, remote) + 1`.

2. **Idempotent replication.** A change with the same `(key, timestamp, node_id)` is applied at most once per node, enforced by `_seen`.

3. **Tombstones are permanent until overwritten.** A delete sets `is_tombstone=True`; `get()` returns `None` for tombstoned keys. Only a subsequent `put` can revive a key.

4. **Pending changes are consumed exactly once.** `get_pending_changes()` returns the list and replaces it with `[]`. No change is sent twice from the same node (though it may be re-queued by a downstream node in ring topology).

5. **Custom merge produces a new timestamp.** When `CUSTOM_MERGE` resolves a conflict, it creates `new_ts = max(local_ts, remote_ts) + 1` and a `canonical_origin = max(local_origin, remote_node)`. This ensures the merged result supersedes both inputs in any subsequent LWW comparison.

6. **ALL_TO_ALL converges in one sync round** (assuming no new writes during sync). Ring requires up to N-1 rounds.

## Error Handling

Minimal — appropriate for a reference implementation:

- `apply_remote_change` raises `ValueError` for unknown `ConflictStrategy` values.
- `sync_until_converged` raises `RuntimeError` if convergence isn't reached within `max_rounds`.
- No validation on inputs to `put`, `get`, or `delete`. Keys and values can be anything hashable/storable.
- `merge_fn` is called without protection — if it raises, the exception propagates through `apply_remote_change` and `sync()`.

## Topics to Explore

- [file] `multi-leader-replication/test_multi_leader.py` — See how concurrent writes, ring convergence, and custom merge functions are exercised in tests
- [file] `conflict-free-replicated-data-types/crdts.py` — CRDTs are the "conflict-free" alternative to the explicit conflict resolution used here; compare the trade-offs
- [file] `vector-clocks/vector_clock.py` — Vector clocks detect causality more precisely than the scalar Lamport clock used here; understand what this implementation trades away
- [function] `leader-follower-replication/replication.py:ReplicaNode` — Compare single-leader replication (no conflicts by design) with this multi-leader approach
- [general] `ring-topology-propagation` — How `_pending` re-queueing in `apply_remote_change` enables multi-hop propagation, and why idempotency via `_seen` is essential to prevent infinite loops

## Beliefs

- `multi-leader-lww-deterministic-tiebreak` — LWW conflict resolution compares `(timestamp, node_id)` tuples, guaranteeing all nodes independently reach the same winner without coordination
- `multi-leader-tombstone-delete` — Deletes are implemented as tombstone writes that replicate like normal mutations; `get()` returns `None` for tombstoned keys
- `multi-leader-custom-merge-new-timestamp` — Custom merge resolution creates a new timestamp (`max(local_ts, remote_ts) + 1`) so the merged result supersedes both conflicting inputs in any future comparison
- `multi-leader-idempotent-apply` — Each `(key, timestamp, origin_node)` triple is applied at most once per node, tracked by the `_seen` set, preventing duplicate application in ring topologies
- `multi-leader-sync-collects-then-distributes` — `sync()` drains all pending queues before distributing any changes, preventing intra-round cascading where one node's change would trigger further propagation within the same sync round

