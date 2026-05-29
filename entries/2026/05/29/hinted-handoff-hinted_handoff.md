# File: hinted-handoff/hinted_handoff.py

**Date:** 2026-05-29
**Time:** 08:56

# Hinted Handoff — `hinted_handoff.py`

## Purpose

This file implements **hinted handoff**, a technique from Dynamo-style leaderless replication systems (DDIA Chapter 5). When a write's preferred replica is temporarily unavailable, another node stores the write as a "hint" and replays it once the target recovers. This lets the system maintain write availability during partial failures without permanently re-routing data to the wrong nodes.

The file owns the full lifecycle: routing writes to preferred replicas, falling back to sloppy quorum via hints, expiring stale hints, and delivering hints when nodes recover.

## Key Components

### `Hint`
A write record destined for a node that was down at write time. It's a pure data object — target node, key/value/version, creation timestamp, and TTL. The only behavior is `is_expired()`, which checks `current_time >= created_at + ttl` (a closed boundary — hints expire exactly at the TTL, not after).

### `Node`
A storage node with two concerns:
- **Main store** (`self.store`): a `dict` mapping keys to `(value, version)` tuples. `put()` uses last-writer-wins by version — it only overwrites if the incoming version is `>=` the existing one.
- **Hint log** (`self.hints`): a flat list of `Hint` objects this node is holding on behalf of other nodes. A node stores hints for *other* nodes, not for itself.

Availability is a simple boolean flag toggled externally — this is a simulation, not a network layer.

### `HintedHandoffStore`
The coordinator. It doesn't represent a single node — it's an omniscient cluster-level controller (appropriate for a teaching implementation). Key responsibilities:

- **`get_preferred_nodes(key)`** — deterministic replica placement via `hash(key) % N` on a sorted node list. Returns `replication_factor` node IDs by walking the ring from the hash position.

- **`put(key, value, current_time)`** — the core write path:
  1. Bump a global version counter for the key.
  2. Try each preferred replica; collect successes and failures.
  3. If `sloppy_quorum=False` and not enough preferred nodes responded, fail immediately.
  4. For each unavailable preferred node, find the first available non-preferred node and store a hint there.
  5. Count `replicas_written + hints_stored` against `write_quorum` to determine success.

- **`get(key)`** — reads only from available *preferred* replicas (no sloppy reads). Returns the highest-versioned value across responding replicas.

- **`trigger_handoff(recovered_node_id, current_time)`** — scans all other nodes for hints targeting the recovered node, replays non-expired ones via `put()`, and purges the hint log. Expired hints are counted but discarded.

## Patterns

**Sloppy quorum** — The `sloppy_quorum` flag controls whether hint-holding nodes count toward the write quorum. When `True` (default), the system trades consistency for availability: a write "succeeds" even if the data only reached non-preferred nodes as hints. When `False`, only actual preferred-replica writes count, and hints are stored optimistically but don't affect the success/failure decision.

**Coordinator pattern** — `HintedHandoffStore` is a god-object coordinator, not a peer in a distributed protocol. This is deliberate for a reference implementation — it avoids the complexity of message passing while demonstrating the algorithm clearly.

**Version-gated writes** — `Node.put()` accepts a write only if `version >= existing_version`. This is a simplified last-writer-wins conflict resolution. The coordinator manages version assignment centrally (monotonic per key), so conflicts can't arise in the normal path — but during handoff replay, stale hints are harmlessly absorbed because their versions will be `<=` the current version.

**Time as a parameter** — All time-dependent operations (`put`, `trigger_handoff`, `expire_all_hints`) take `current_time` as an explicit argument rather than calling `time.time()`. This makes the implementation fully deterministic and testable.

## Dependencies

**Imports**: None — the module is entirely self-contained with no external dependencies.

**Imported by**:
- `test_hinted_handoff.py` — unit/integration tests
- `tester_test_hinted_handoff.py` — likely a test validation harness

## Flow

### Write path (`put`)

```
put(key, value, current_time)
  │
  ├─ bump version counter
  ├─ get_preferred_nodes(key) → [A, B, C]
  │
  ├─ for each preferred node:
  │     available? → node.put() → add to replicas_written
  │     unavailable? → add to unavailable_preferred
  │
  ├─ if strict quorum and replicas_written < write_quorum → FAIL
  │
  ├─ for each unavailable preferred node:
  │     find first available non-preferred node
  │     store Hint on that node
  │     add to hints_stored
  │
  └─ success = (replicas_written + hints_stored) >= write_quorum
```

### Handoff path (`trigger_handoff`)

```
trigger_handoff(recovered_node_id, current_time)
  │
  ├─ for every OTHER node in the cluster:
  │     get hints targeting recovered_node_id
  │     for each hint:
  │       expired? → count as expired, skip
  │       valid? → recovered_node.put(key, value, version)
  │     remove all hints for recovered_node_id
  │
  └─ return { hints_delivered, hints_expired, keys_recovered }
```

## Invariants

1. **One hint per unavailable target per write** — for each unavailable preferred node, only *one* non-preferred node stores the hint (the first available one found). This prevents hint explosion.

2. **Hints live on non-preferred nodes only** — the hint-storing loop explicitly excludes preferred nodes from the candidate set. A preferred node never stores hints for a peer in the same preference list.

3. **Version monotonicity per key** — `self.versions[key]` is incremented on every `put()` call, so versions are globally monotonic per key. This means handoff replays of older writes are safely absorbed by the version check in `Node.put()`.

4. **Reads never consult non-preferred nodes** — `get()` only queries preferred replicas. This means data stored only as hints is invisible to reads until handoff completes. This is a deliberate trade-off: sloppy quorum improves write availability but not read availability.

5. **Hint cleanup is all-or-nothing per target** — `remove_hints_for()` purges all hints for a target regardless of whether individual hints succeeded or expired. After `trigger_handoff`, no hints for that node remain on any other node.

## Error Handling

There is essentially none, by design. The module signals write failure through the `success: False` return value — it never raises exceptions. Invalid node IDs would raise `KeyError` from the `self.nodes` dict, but this is an unchecked precondition rather than intentional error handling. The `is_expired` check silently discards stale hints during handoff rather than surfacing them as errors.

## Topics to Explore

- [file] `hinted-handoff/test_hinted_handoff.py` — See how sloppy vs. strict quorum, hint expiry, and multi-node failure scenarios are tested
- [file] `leaderless-replication/dynamo.py` — The broader Dynamo-style replication system that hinted handoff plugs into in production architectures
- [file] `read-repair/read_repair.py` — The complementary anti-entropy mechanism: read-repair fixes stale replicas at read time, while hinted handoff fixes them at recovery time
- [function] `consistent-hashing/consistent_hashing.py:ConsistentHashRing` — How production systems do replica placement with virtual nodes instead of the `hash % N` approach used here
- [general] `sloppy-quorum-tradeoffs` — DDIA Section 5.4.2 discusses how sloppy quorums provide durability but not the consistency guarantees of strict quorums — a write that "succeeds" via hints may not be readable until handoff completes

## Beliefs

- `hinted-handoff-one-hint-per-target-per-write` — Each unavailable preferred replica gets at most one hint stored on one non-preferred node per write operation
- `hinted-handoff-reads-skip-non-preferred` — `get()` only queries preferred replicas, so data existing only as hints is invisible to reads until handoff delivers it
- `hinted-handoff-sloppy-quorum-counts-hints` — When `sloppy_quorum=True`, stored hints count toward `write_quorum` the same as direct replica writes
- `hinted-handoff-version-monotonic-per-key` — The coordinator assigns strictly increasing versions per key, so stale hint replays are absorbed harmlessly by `Node.put()`'s `>=` check
- `hinted-handoff-no-exceptions` — Write failures are signaled via `success: False` in the return dict; the module never raises exceptions in normal operation

