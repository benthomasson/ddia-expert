# Function: MultiLeaderCluster in multi-leader-replication/multi_leader.py

**Date:** 2026-05-29
**Time:** 14:13

# `MultiLeaderCluster`

## Purpose

`MultiLeaderCluster` is a simulation harness for multi-leader (multi-master) replication, the architecture described in DDIA Chapter 5 where multiple nodes independently accept writes and asynchronously replicate changes to each other. It owns a set of `ReplicaNode` instances, wires them together according to a chosen replication topology, and drives the sync protocol that propagates pending writes between nodes.

The class exists to make the replication topology and conflict resolution strategy configurable at the cluster level, so callers can write to any node directly and then call `sync()` to simulate an asynchronous replication round — without manually wiring up which node talks to which.

## Contract

**Preconditions:**
- `node_ids` must be non-empty and contain unique IDs (duplicates silently overwrite earlier nodes in the dict).
- If `strategy` is `CUSTOM_MERGE`, then `merge_fn` must be a valid callable with the signature `(key, local_value, remote_value, local_ts, remote_ts) -> merged_value`. This is not enforced at construction — it will blow up at `apply_remote_change` time when the first conflict occurs.

**Postconditions:**
- After `sync()`, all pending changes from every node have been delivered to their replication targets (all peers for ALL_TO_ALL, next neighbor for RING).
- After `sync_until_converged()` returns successfully, `all_converged()` is `True`.

**Invariants:**
- Node order is stable (`_node_order` is a list, not derived from dict key order), which matters for RING topology where the neighbor relationship is positional.
- `sync()` is snapshot-isolated within a round: it collects all pending changes *before* delivering any, so a write propagated in round N doesn't cascade further within the same round.

## Parameters

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `node_ids` | `list[str]` | — | Identifiers for each replica node. Order defines the ring for `RING` topology. |
| `strategy` | `ConflictStrategy` | `LAST_WRITE_WINS` | How concurrent writes to the same key are resolved. LWW compares `(timestamp, node_id)` tuples lexicographically. |
| `merge_fn` | `Optional[Callable]` | `None` | User-supplied merge function, required when `strategy` is `CUSTOM_MERGE`. Ignored otherwise. |
| `topology` | `Topology` | `ALL_TO_ALL` | Replication graph shape. ALL_TO_ALL sends every change to every other node per round. RING sends only to the next node in the list. |

**Edge cases:**
- A single-node cluster works fine — `sync()` returns 0, `all_converged()` returns `True`.
- Empty `node_ids` would create a cluster with no nodes; `sync()` returns 0, `all_converged()` returns `True` (falls through the `<= 1` check). Not explicitly guarded.

## Return Values

| Method | Returns | Conditions |
|--------|---------|------------|
| `node(id)` | `ReplicaNode` | Raises `KeyError` if `id` is unknown. |
| `sync()` | `int` — number of change deliveries attempted | Counts every `apply_remote_change` call, including idempotent no-ops (where the change was already seen). |
| `all_converged()` | `bool` | `True` when all nodes have identical key sets and matching `(value, timestamp, is_tombstone)` tuples for every key. |
| `sync_until_converged()` | `int` — rounds taken | Raises `RuntimeError` if not converged after `max_rounds`. |

## Algorithm

### `sync()`

1. **Snapshot pending changes**: Iterates `_node_order` and calls `get_pending_changes()` on each node. This drains and returns each node's outbound log, so the same change is never sent twice across rounds.

2. **Deliver based on topology**:
   - **ALL_TO_ALL**: For each source node, each pending change is delivered to every *other* node via `apply_remote_change`. A change from node A reaches B and C in the same round.
   - **RING**: Each node sends only to its successor (`(i+1) % len`). A change from node 0 reaches node 1 this round, then node 1 will forward it to node 2 *next* round (because `apply_remote_change` enqueues it in node 1's pending list, but that list was already drained for this round).

3. The return count is the number of `apply_remote_change` calls, not the number of *accepted* changes. Idempotent skips inside `apply_remote_change` (via the `_seen` set) are still counted.

### `all_converged()`

1. Takes the first node as the reference.
2. Checks that all other nodes have the same set of keys.
3. For each key, compares `(value, timestamp, is_tombstone)` — tuple indices `(0, 1, 3)`. Notably skips index 2 (`origin_node_id`), because different nodes may record different origins for the same resolved value depending on arrival order.

### `sync_until_converged()`

Calls `sync()` then `all_converged()` in a loop up to `max_rounds` times. For ALL_TO_ALL, one round suffices if there are no custom-merge cascades. For RING with N nodes, convergence takes at least N−1 rounds because each hop only reaches the next neighbor.

## Side Effects

- **`sync()` drains every node's pending queue** — calling `get_pending_changes()` is destructive (it clears `_pending`). Calling `sync()` twice without intervening writes produces zero propagations on the second call.
- **`apply_remote_change` mutates node state**: updates `_store`, `_clock`, `_seen`, `_pending`, and `_conflict_log` on each destination node.
- **CUSTOM_MERGE generates new changes**: the merge result is enqueued in `_pending` with a synthetic timestamp and canonical origin, meaning the merged value will propagate further in subsequent rounds.

## Error Handling

- `node()` raises `KeyError` for unknown node IDs (dict lookup, no wrapping).
- `sync_until_converged()` raises `RuntimeError` if convergence isn't reached within `max_rounds`.
- If `strategy` is `CUSTOM_MERGE` and `merge_fn` is `None`, a `TypeError` will propagate from within `apply_remote_change` when the first conflict triggers `merge_fn(...)`.
- No validation on `node_ids` — duplicate IDs, empty lists, or non-string values are accepted silently.

## Usage Patterns

```python
cluster = MultiLeaderCluster(["dc1", "dc2", "dc3"])

# Write to different leaders concurrently
cluster.node("dc1").put("user:1", "Alice")
cluster.node("dc2").put("user:1", "Bob")  # concurrent write → conflict

# Replicate
cluster.sync()

# Check convergence
assert cluster.all_converged()

# Or just sync until done (useful for RING topology)
cluster = MultiLeaderCluster(["a", "b", "c"], topology=Topology.RING)
cluster.node("a").put("x", 1)
rounds = cluster.sync_until_converged()  # will take 2 rounds for 3 nodes
```

**Caller obligations:**
- Call `sync()` after writes to propagate them. Writes are purely local until synced.
- For RING topology, understand that convergence takes multiple rounds proportional to cluster size.
- Supply `merge_fn` when using `CUSTOM_MERGE`, or the first conflict will crash.

## Dependencies

- **`ReplicaNode`** (same module): the actual replica with Lamport clock, store, conflict detection, and the `_seen` deduplication set.
- **`ConflictStrategy`** and **`Topology`** enums (same module): configuration discriminators.
- **`ConflictRecord`** dataclass (same module): returned by `apply_remote_change` on conflict, accumulated in each node's `_conflict_log`.
- No external dependencies — pure Python, no I/O, fully deterministic given the same call sequence.

## Topics to Explore

- [function] `multi-leader-replication/multi_leader.py:ReplicaNode.apply_remote_change` — The core conflict detection and resolution logic; understanding how `_seen` provides idempotency and how LWW vs custom merge differ
- [file] `multi-leader-replication/test_multi_leader.py` — Test cases that exercise concurrent writes, ring convergence timing, tombstone propagation, and custom merge functions
- [general] `ring-topology-convergence` — How many sync rounds RING requires for N nodes, and how `apply_remote_change` re-enqueuing into `_pending` enables store-and-forward propagation
- [function] `multi-leader-replication/multi_leader.py:ReplicaNode._record_seen` — The idempotency mechanism that prevents duplicate application of the same change, keyed on `(timestamp, origin)` pairs
- [general] `custom-merge-cascade-behavior` — How `CUSTOM_MERGE` generates synthetic changes with new timestamps and canonical origins, and whether this can cause infinite convergence loops

## Beliefs

- `multi-leader-sync-snapshot-isolation` — `sync()` drains all nodes' pending queues before delivering any changes, preventing intra-round cascading; a change propagated in round N cannot cascade further until round N+1
- `multi-leader-ring-convergence-rounds` — RING topology requires at least N−1 `sync()` rounds to fully propagate a single-source change across N nodes, because each round advances the change by exactly one hop
- `multi-leader-convergence-skips-origin` — `all_converged()` compares `(value, timestamp, is_tombstone)` but intentionally ignores `origin_node_id` (tuple index 2), since different arrival orders can record different origins for the same resolved state
- `multi-leader-sync-count-includes-noops` — The count returned by `sync()` includes idempotent no-op deliveries (changes already in the target's `_seen` set), so it reflects replication *attempts*, not accepted changes
- `multi-leader-custom-merge-requires-merge-fn` — Constructing a cluster with `CUSTOM_MERGE` strategy and `merge_fn=None` is accepted silently, but will raise `TypeError` at the first actual conflict during `sync()`

