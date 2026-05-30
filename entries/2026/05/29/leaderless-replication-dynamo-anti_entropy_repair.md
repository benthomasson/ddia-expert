# Function: anti_entropy_repair in leaderless-replication/dynamo.py

**Date:** 2026-05-29
**Time:** 13:34

# `anti_entropy_repair` — Dynamo-style Anti-Entropy Repair

## Purpose

`anti_entropy_repair` is a background reconciliation mechanism that brings all available replicas into a consistent state. In Dynamo-style systems, replicas can diverge — a node might have been down during a write, or a write might have reached only the quorum subset. While **read repair** (in `get()`) fixes divergence lazily on read, anti-entropy repair fixes it proactively across the *entire* keyspace, regardless of whether any client reads those keys.

This is the implementation of the concept from DDIA Chapter 5: in leaderless replication, anti-entropy processes run in the background to detect and repair differences between replicas that read repair alone would miss (e.g., keys that are rarely read).

## Contract

- **Precondition**: The cluster must have nodes initialized. No locking or coordination is assumed — this is designed to run when the cluster is otherwise quiescent.
- **Postcondition**: After completion, every available node holds the highest-versioned value for every key known to any available node. Unavailable nodes are untouched and remain divergent.
- **Invariant**: The method never creates new versions — it only propagates existing values. Version counters (`_version_counters`) are not modified.

## Parameters

None (it's a method on `DynamoCluster` using `self`).

## Return Value

Returns an `int` — the number of individual node-key repairs performed. A repair is counted each time a node is written to because it was missing a key or held a stale version. If all nodes are already consistent, returns `0`.

Note: the count can exceed the number of keys. If 3 nodes are behind on the same key, that's 3 repairs.

## Algorithm

**Phase 1 — Key discovery:**
Iterates all available nodes, collecting the union of all keys from their stores into `all_keys`. Unavailable nodes are skipped entirely — their keys won't be discovered and won't be repaired.

**Phase 2 — Per-key reconciliation (two sub-passes per key):**

1. **Find best version**: Scans every available node's store for the key, tracking the `VersionedValue` with the highest `version` number. If no node has the key (shouldn't happen given Phase 1, but guarded), skips to the next key.

2. **Push repairs**: Scans every available node again. If a node is missing the key (`current is None`) or has a version strictly less than the best, writes the best value to that node via `node.write()`. Increments the repair counter for each such write.

## Side Effects

- **Mutates replica stores**: Directly calls `node.write(key, best.value, best.version)` on stale replicas, updating their `_store` dictionaries.
- **No version counter mutation**: Unlike `put()`, this does not touch `self._version_counters`. It propagates existing versions, not new ones.
- **No hints generated**: Does not interact with the hinted handoff system.

## Error Handling

None. The method does not raise exceptions. If all nodes are unavailable, `all_keys` will be empty and it returns `0`. The `node.write()` call can't fail here because availability is checked before each write (and `write()` returns `False` for unavailable nodes, though that path shouldn't be hit).

## Usage Patterns

Typically called periodically or after a node recovery event:

```python
cluster.set_node_available("node_1", True)  # node comes back up
cluster.deliver_hints()                      # replay buffered writes
repaired = cluster.anti_entropy_repair()     # catch anything hints missed
```

Anti-entropy is complementary to read repair and hinted handoff — it's the "sweep" that catches anything the other two mechanisms missed. In tests, it's called to verify convergence after failure scenarios.

## Dependencies

- `ReplicaNode.write()` — the write-if-not-stale method (accepts `version >= current`)
- `ReplicaNode._store` — directly accessed (bypassing the `read()` method) to avoid the availability check on reads during the discovery phase. Note: the availability *is* checked via `node.is_available` before accessing `_store`, so this isn't actually bypassing a meaningful guard — it's just avoiding the `None` return from `read()` for keys that exist.

## Assumptions Not Enforced by Types

1. **Single-writer version scheme**: The algorithm assumes version numbers are globally ordered and that the highest version is always the "correct" value. It does not handle concurrent writes that produce the same version number with different values (true conflicts). The `put()` method uses a global counter, so this holds within this implementation — but it's an architectural assumption, not a type constraint.

2. **Availability is stable during repair**: The method checks `is_available` at each access, but a node could become unavailable mid-repair. The two-pass structure (find best, then push) means a node could be available during discovery but unavailable during the push, or vice versa. This is benign — the worst case is a missed repair, caught next time.

3. **No concurrent writes during repair**: There's no locking. A concurrent `put()` could advance a key's version between the "find best" and "push" passes, causing the repair to write a now-stale value. The `write()` method's `version >= current` guard prevents regression, so this is safe but could cause a redundant write.

---

## Topics to Explore

- [function] `leaderless-replication/dynamo.py:deliver_hints` — The other convergence mechanism; compare how hinted handoff targets specific missed writes vs. anti-entropy's full-keyspace sweep
- [function] `leaderless-replication/dynamo.py:get` — Contains the read-repair logic that provides lazy convergence; anti-entropy is the eager counterpart
- [file] `leaderless-replication/test_dynamo.py` — Test scenarios that exercise anti-entropy after node failures, showing the intended usage patterns
- [general] `merkle-tree-anti-entropy` — Real Dynamo/Cassandra systems use Merkle trees to efficiently detect divergence instead of scanning every key; this implementation does the naive full scan
- [function] `leaderless-replication/dynamo.py:put` — Understanding the version counter scheme is essential to understanding why "highest version wins" is safe here

## Beliefs

- `anti-entropy-skips-unavailable-nodes` — Anti-entropy repair only reads from and writes to nodes where `is_available` is true; down nodes are completely excluded from both key discovery and repair propagation
- `anti-entropy-does-not-advance-versions` — The repair method propagates existing versioned values without modifying `_version_counters`, so it never creates new versions
- `anti-entropy-counts-per-node-writes` — The returned repair count reflects individual node writes, not unique keys repaired; one stale key across N nodes counts as N repairs
- `anti-entropy-complements-read-repair` — Read repair in `get()` fixes divergence lazily per-key on read; anti-entropy proactively sweeps the full keyspace to catch keys that are never read
- `anti-entropy-highest-version-wins` — The repair assumes a total ordering on versions where the highest version number is authoritative, which is safe because `put()` uses a global per-key version counter

