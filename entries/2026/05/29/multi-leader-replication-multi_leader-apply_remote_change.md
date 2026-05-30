# Function: apply_remote_change in multi-leader-replication/multi_leader.py

**Date:** 2026-05-29
**Time:** 14:16

# `apply_remote_change` — Multi-Leader Conflict Resolution

## Purpose

This is the inbound replication handler for a multi-leader (multi-master) system. When a replica receives a write that originated on another node, this method decides whether the incoming change conflicts with local state and, if so, resolves the conflict using a pluggable strategy (last-write-wins or custom merge). It's the core of the convergence mechanism — without it, replicas would diverge permanently after concurrent writes.

## Contract

**Preconditions:**
- `change` must contain keys `"key"`, `"timestamp"`, `"node_id"`, `"value"`, and optionally `"is_tombstone"`.
- If `strategy` is `CUSTOM_MERGE`, `merge_fn` must be non-None and callable with signature `(key, local_val, remote_val, local_ts, remote_ts) -> merged_value`. The code does **not** guard against `merge_fn` being `None` — it will raise `TypeError` at call time.
- Timestamps are Lamport clocks (monotonic integers), not wall-clock times.

**Postconditions:**
- The node's Lamport clock is advanced to at least `max(local_clock, remote_ts) + 1`.
- The `(remote_ts, remote_node)` pair is recorded in `_seen[key]`, guaranteeing idempotency on replay.
- If a conflict occurred, a `ConflictRecord` is appended to `_conflict_log` and returned.
- If the change was applied (not deduplicated), it's queued in `_pending` for downstream propagation.

**Invariant:** Calling `apply_remote_change` twice with the same `change` dict is a no-op after the first application — the `_seen` set enforces exactly-once semantics per `(key, timestamp, node_id)` triple.

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `change` | `dict` | Replication log entry from another node. Must have `key`, `timestamp`, `node_id`, `value`; `is_tombstone` defaults to `False`. |
| `strategy` | `ConflictStrategy` | How to resolve conflicts. `LAST_WRITE_WINS` picks the higher `(timestamp, node_id)` tuple. `CUSTOM_MERGE` delegates to `merge_fn`. |
| `merge_fn` | `Optional[Callable]` | Required when `strategy` is `CUSTOM_MERGE`. Receives `(key, local_value, remote_value, local_ts, remote_ts)` and returns the merged value. Ignored otherwise. |

**Edge cases on `change`:** Tombstoned values arrive with `is_tombstone: True` and `value: None`. The method stores the sentinel `_TOMBSTONE` object internally but reports `None` in `ConflictRecord.remote_value` for tombstones — the caller sees deletion as `None`, not the sentinel.

## Return Value

- **`None`** — No conflict. Either the change was a duplicate (idempotency), the key didn't exist locally, or the remote change came from the same origin (sequential update, not concurrent write).
- **`ConflictRecord`** — A conflict was detected and resolved. The record captures both values, both timestamps, the resolved value, and which strategy resolved it. The caller can log this, surface it to the user, or ignore it — the store is already updated.

## Algorithm

1. **Dedup check.** Look up `(remote_ts, remote_node)` in `_seen[key]`. If present, this is a replay — return `None` immediately. This is critical for ring topologies where changes circulate through every node.

2. **Advance Lamport clock.** Set `_clock = max(_clock, remote_ts) + 1`. This ensures causal ordering: any subsequent local write will have a timestamp strictly greater than the remote one.

3. **Record seen.** Mark this `(ts, node)` pair as processed for future dedup.

4. **No local entry.** If the key doesn't exist in `_store`, accept unconditionally. Store the value (or `_TOMBSTONE`), queue for propagation, return `None`.

5. **Same-origin, same-timestamp.** If local and remote have identical `(origin, timestamp)`, this is the same write arriving via a different path — skip it.

6. **Same origin, different timestamp.** The remote node sent a newer version of its own write. This is a sequential update, not a conflict. Accept it if the remote `(ts, node_id)` tuple is lexicographically greater. Always propagate.

7. **Different origins — conflict.** Two nodes wrote to the same key concurrently. Resolve:
   - **LWW:** Compare `(timestamp, node_id)` tuples. Higher wins. The node_id tiebreaker ensures deterministic resolution across all replicas even when timestamps collide.
   - **Custom merge:** Call `merge_fn`, assign a new timestamp (`max + 1`), pick a canonical origin (`max(local_origin, remote_node)` — deterministic), and propagate the **merged** result as a new change. This is the only strategy that synthesizes a new change rather than forwarding the original.

8. **Log and return.** Append the `ConflictRecord` to `_conflict_log`. For LWW, forward the original change. For custom merge, the merged change was already appended in step 7. Return the record.

## Side Effects

- **Mutates `_store`**: May overwrite the value for `key`.
- **Mutates `_clock`**: Always advances the Lamport clock.
- **Mutates `_seen`**: Always records the incoming `(ts, node)` pair.
- **Mutates `_pending`**: Appends the change (or a synthesized merged change) for downstream propagation. This is how changes ripple through ring topologies.
- **Mutates `_conflict_log`**: Appends a record when a conflict is resolved.
- **No I/O, no locking.** This is a pure in-memory operation — thread safety is the caller's responsibility.

## Error Handling

- **`ValueError`**: Raised if `strategy` is not a recognized `ConflictStrategy` variant. This is a programmer error, not a runtime condition.
- **`TypeError` (implicit)**: If `strategy` is `CUSTOM_MERGE` and `merge_fn` is `None`, Python will raise `TypeError: 'NoneType' object is not callable`. The code trusts the caller to pair these correctly.
- **`KeyError` (implicit)**: If `change` is missing required keys (`"key"`, `"timestamp"`, etc.), Python raises `KeyError`. No defensive checking — the dict schema is an assumed contract.

## Usage Patterns

Typical caller is `MultiLeaderCluster.sync()`, which collects pending changes from all nodes, then applies them to destination nodes based on topology:

```python
# All-to-all: every change goes to every other node
for change in pending_by_node[src_id]:
    for dst_id in self._node_order:
        if dst_id != src_id:
            self._nodes[dst_id].apply_remote_change(change, strategy, merge_fn)
```

**Caller obligations:**
- Must call `get_pending_changes()` before `apply_remote_change()` in the same sync round to avoid infinite loops where a node's own propagated changes get re-applied.
- Must provide a valid `merge_fn` when using `CUSTOM_MERGE`.
- Should call `sync()` repeatedly (or use `sync_until_converged()`) since ring topologies require multiple rounds — a change must hop through every node before convergence.

## Dependencies

- **`_TOMBSTONE`**: Module-level sentinel object. Identity-compared, never exposed to callers — `get()` returns `None` for tombstoned keys.
- **`ConflictStrategy`**: Enum controlling resolution. Only two variants; adding a third requires updating this method.
- **`ConflictRecord`**: Dataclass for conflict metadata. Pure data, no behavior.
- **Lamport clock (`_clock`)**: Shared mutable state on the node. Both `put()` and `apply_remote_change()` advance it, so interleaving local writes with remote applies maintains causal ordering.

## Assumptions Not Enforced by Types

1. **Timestamps are positive integers from Lamport clocks**, not arbitrary values. Negative or zero timestamps would break the `max + 1` advancement.
2. **Node IDs are comparable strings** — they're used in tuple comparison `(ts, node_id)` for tiebreaking and in `max(local_origin, remote_node)` for canonical origin selection.
3. **`change` dicts are not mutated after being passed in.** The same dict may be forwarded to `_pending` and later applied to other nodes — mutation would corrupt the replication log.
4. **Single-threaded execution.** No synchronization on `_store`, `_pending`, `_seen`, or `_clock`.

---

## Topics to Explore

- [function] `multi-leader-replication/multi_leader.py:sync` — Orchestrates replication rounds; understanding it reveals how `_pending` drives convergence and why ring topology needs multiple rounds
- [file] `multi-leader-replication/test_multi_leader.py` — Test cases show the conflict scenarios this code handles: concurrent writes, tombstone conflicts, custom merge functions, ring propagation
- [function] `multi-leader-replication/multi_leader.py:all_converged` — The convergence checker that validates whether all replicas agree; exposes what "converged" actually means (value + timestamp + tombstone flag)
- [general] `lamport-clock-ordering` — How Lamport timestamps provide causal ordering without synchronized clocks, and why the `(timestamp, node_id)` tuple gives a total order
- [general] `tombstone-replication-semantics` — How deletes propagate as tombstones in multi-leader systems and why `_TOMBSTONE` sentinel vs. key removal matters for conflict detection

## Beliefs

- `apply-remote-change-idempotent` — `apply_remote_change` is idempotent: replaying the same `(key, timestamp, node_id)` triple is silently skipped via the `_seen` set
- `lww-tiebreak-uses-node-id` — Last-write-wins resolves timestamp ties deterministically by comparing `(timestamp, node_id)` tuples lexicographically, so all replicas pick the same winner
- `custom-merge-synthesizes-new-change` — `CUSTOM_MERGE` creates a new change with `timestamp = max(local, remote) + 1` and propagates the merged result, unlike LWW which forwards the original change
- `conflict-requires-different-origins` — A conflict is only detected when `local_origin != remote_node`; two writes from the same origin at different timestamps are treated as sequential updates, not conflicts
- `pending-queue-drives-ring-propagation` — Every accepted remote change is appended to `_pending`, enabling multi-hop propagation in ring topologies where changes must traverse all nodes to converge

