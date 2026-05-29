# Topic: How `_pending` re-queueing in `apply_remote_change` enables multi-hop propagation, and why idempotency via `_seen` is essential to prevent infinite loops

**Date:** 2026-05-29
**Time:** 09:05

# Multi-Hop Propagation via `_pending` and Idempotency via `_seen`

## The Core Mechanism

In a multi-leader replication cluster, a write originating at node A must eventually reach *every* node — not just A's direct neighbors. In a ring topology, node A only talks to node B, which talks to node C, and so on. The write has to "hop" through intermediaries. The `_pending` queue and `_seen` set work together to make this happen safely.

### How `_pending` re-queueing enables forwarding

When a node receives a remote change via `apply_remote_change` (`multi-leader-replication/multi_leader.py:94`), it doesn't just apply the change locally — it **re-enqueues the change into its own `_pending` list** so it will be forwarded to the next node during the next replication cycle.

This happens in three places within `apply_remote_change`:

1. **No local value exists** (line 118): The node accepts the change and immediately appends it to `_pending`:
   ```python
   self._pending.append(change)
   ```

2. **Same-origin update, no conflict** (line 136): The newer version wins, and the change is re-queued:
   ```python
   self._pending.append(change)
   ```

3. **Conflict with custom merge** (line 170): After merging, a *new* change (with the merged value, a fresh timestamp, and a canonical origin) is appended to `_pending`.

4. **Conflict with LWW** (line 192, the final `self._pending.append(change)` after the conflict block): The original change is forwarded as-is.

The `get_pending_changes` method (line 88) atomically drains this queue — returning all accumulated changes and resetting `_pending` to empty. The `MultiLeaderCluster` orchestrator then delivers these drained changes to the next node(s) in the topology.

**This is the multi-hop engine**: Node A writes → change lands in A's `_pending` → cluster delivers to B → B's `apply_remote_change` applies it *and* re-enqueues to B's `_pending` → cluster delivers to C → and so on until every node has the change.

### Why `_seen` is essential

Without idempotency, this re-queueing creates infinite loops. Consider a three-node ring A → B → C → A:

1. A writes key `"x"` at timestamp 1
2. A's change propagates to B, B re-queues it
3. B's change propagates to C, C re-queues it
4. C's change propagates back to **A** — and without `_seen`, A would re-queue it again
5. A propagates to B again, B re-queues... **infinite loop**

The `_seen` dictionary (`multi_leader.py:42`) prevents this:

```python
self._seen: dict[str, set[tuple[int, str]]] = {}
```

It maps each key to a set of `(timestamp, origin_node_id)` pairs that this node has already processed. The idempotency check at line 104 is the **first thing** `apply_remote_change` does after extracting the change fields:

```python
if key in self._seen and (remote_ts, remote_node) in self._seen[key]:
    return None
```

When node A originally wrote key `"x"`, it called `_record_seen(key, ts, self.node_id)` at line 57. So when C's forwarded copy of that same change arrives back at A, the `(timestamp=1, origin="A")` pair is already in A's `_seen["x"]` set. The change is silently dropped — no re-queueing, no conflict resolution, no further propagation. The loop dies.

### The `_record_seen` helper

Every path that accepts a change calls `_record_seen` (line 48–51):

```python
def _record_seen(self, key: str, ts: int, origin: str):
    if key not in self._seen:
        self._seen[key] = set()
    self._seen[key].add((ts, origin))
```

This is called:
- On local `put` (line 57) and `delete` (line 78) — so the originator is immune to its own writes bouncing back
- On accepted remote changes (line 109) — so a forwarded change is only processed once per node

### The custom-merge complication

When a conflict is resolved via `CUSTOM_MERGE` (lines 160–177), the merged result gets a **new timestamp and canonical origin**. This means the merged change is a genuinely new `(ts, origin)` pair that no node has seen before, so it will propagate through the full ring without being dropped. This is correct — every node needs the merged result, not the original conflicting values.

## Topics to Explore

- [function] `multi-leader-replication/multi_leader.py:MultiLeaderCluster` — The orchestrator that calls `get_pending_changes` and `apply_remote_change` across nodes; shows how ring vs. all-to-all topology affects which nodes receive forwarded changes
- [file] `conflict-free-replicated-data-types/crdts.py` — CRDTs solve the same convergence problem differently: `merge` is idempotent by mathematical construction (join-semilattice), eliminating the need for explicit `_seen` tracking
- [file] `gossip-protocol/gossip_protocol.py` — Another multi-hop propagation system; compare how `receive_gossip` uses heartbeat counters for idempotency instead of a seen-set, and how bidirectional exchange differs from ring forwarding
- [general] `ring-vs-all-to-all-propagation-delay` — In all-to-all topology, changes reach every node in one hop (no re-queueing needed for coverage); in ring topology, propagation takes N-1 hops — explore how this affects convergence latency and conflict probability
- [function] `multi-leader-replication/multi_leader.py:apply_remote_change` — Trace what happens when two nodes concurrently write the same key in a ring: the LWW path re-queues the original change (loser's write still propagates), while CUSTOM_MERGE re-queues a synthetic merged change

## Beliefs

- `pending-requeue-enables-multihop` — Every accepted remote change in `apply_remote_change` is appended to `_pending`, causing it to be forwarded to the next node in the replication topology
- `seen-set-keyed-by-ts-origin` — `_seen` maps each key to a set of `(timestamp, origin_node_id)` tuples; a change is skipped if its exact `(ts, origin)` pair is already in the set for that key
- `local-writes-self-immunize` — Both `put` and `delete` call `_record_seen` with the local node's ID, ensuring the originating node drops its own changes when they loop back through the ring
- `custom-merge-creates-new-identity` — Conflict resolution via `CUSTOM_MERGE` generates a new `(timestamp, canonical_origin)` pair not in any node's `_seen` set, so the merged result propagates to all nodes as a fresh change
- `get-pending-drains-atomically` — `get_pending_changes` returns the current `_pending` list and replaces it with an empty list, ensuring each change is delivered exactly once per replication cycle

