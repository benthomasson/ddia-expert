# Topic: The Dynamo implementation has no delete/tombstone support; designing one that's safe under read repair and anti-entropy would be a valuable exercise

**Date:** 2026-05-29
**Time:** 09:00

# The Missing Delete: Why Tombstones Matter in Dynamo-Style Replication

## The Problem

Look at `ReplicaNode.write()` in `leaderless-replication/dynamo.py:42-49`: the only mutation it supports is writing a value with a version. There's no `delete()` method on `ReplicaNode`, no `delete()` on `DynamoCluster`, and no concept of a tombstone anywhere in the store. The same is true in the read-repair variant at `read-repair/read_repair.py:21-27` — `Replica.put()` stores values, but there's no way to remove them.

This isn't just a missing convenience method. In a leaderless system, "just remove the key from the dict" is actively dangerous. Here's why.

## Why Naive Deletion Resurrects Data

Suppose you have three replicas and you delete key `"user:42"` by removing it from `_store` on the two nodes that are available. The third node is down. When it comes back up, two repair mechanisms will **resurrect the deleted data**:

**Read repair** (`dynamo.py:139-160`): When `get()` is called, it collects responses from available nodes. If the recovered node still has `"user:42"` at version 5, while the other two return `None` (key missing), the current logic at lines 155-160 finds the max version and pushes it to stale replicas. Since `None` has no version and version 5 > nothing, read repair will **re-write the deleted value back to the nodes that deleted it**.

**Anti-entropy** (`dynamo.py:180-end`): The `anti_entropy_repair()` method iterates all keys across all available nodes, finds the highest version, and pushes it everywhere. If even one node still holds the deleted key, anti-entropy will propagate it back to every other node. The key collection at line 183-185 unions keys from all nodes — a key that exists on any single node will be "repaired" onto every other node.

The same resurrection happens in `read-repair/read_repair.py:143-165` (`anti_entropy_repair`) — it finds the max version across replicas and pushes it to any replica that doesn't have it.

## What Tombstones Solve

A tombstone is a special marker that says "this key was deliberately deleted at version V." It's stored like any other value but with a sentinel (e.g., `value=None` plus a `deleted=True` flag). This means:

1. **Read repair sees the delete as the latest version.** When the recovered node returns version 5 and the other two return a tombstone at version 6, the repair logic correctly pushes the tombstone (the deletion) to the stale node rather than resurrecting the old value.

2. **Anti-entropy propagates deletions.** The tombstone participates in version comparison just like any write. The highest-version entry wins, and if that entry is a tombstone, the key stays deleted.

## The Design Challenge

Making tombstones work safely requires changes in several places:

**`VersionedValue` (`dynamo.py:14-18`)** needs a `deleted` or `is_tombstone` flag. You can't just use `value=None` because `None` is a legitimate "key not found" return from `read()` at line 55.

**`DynamoCluster.get()` (`dynamo.py:120-165`)** must distinguish between "key was never written" (no responses) and "key was deleted" (tombstone is the latest version). The current code at line 135 returns `ReadResult(value=None, version=0)` for both cases — a caller can't tell the difference.

**Read repair (`dynamo.py:150-160`)** must treat tombstones as valid highest-version entries and push them to stale nodes, not skip them as "missing."

**Anti-entropy (`dynamo.py:180-end`)** must propagate tombstones the same way it propagates values. The current logic picks the highest version; tombstones just need to participate in that comparison.

**Tombstone garbage collection** is the hardest part. You can't keep tombstones forever — they'd accumulate without bound. But you can't remove them too early — if a node is down longer than your GC interval, it'll come back with live data that no tombstone exists to suppress, and you're back to resurrection. The hinted handoff implementation (`hinted-handoff/hinted_handoff.py:7-18`) shows one approach: TTL-based expiration. But a tombstone TTL must be longer than the maximum expected node downtime, which is hard to bound.

**Vector clocks** (`vector-clocks/vector_clock.py`) would interact with tombstones in the versioned store variant. The `VersionedKVStore._receive_replica()` method at line 142-155 uses dominance checks to decide whether to keep or drop incoming versions — a tombstone version must dominate and replace the live value for deletion to propagate, and a live value must never supersede a tombstone with a concurrent or later vector clock.

## Topics to Explore

- [function] `leaderless-replication/dynamo.py:anti_entropy_repair` — Trace exactly how this propagates state across nodes, and identify where a tombstone check would need to be inserted to prevent resurrection
- [file] `hinted-handoff/hinted_handoff.py` — The TTL-based hint expiration pattern here is directly analogous to tombstone GC; understanding the tradeoff between hint TTL and node recovery time maps onto the tombstone lifetime problem
- [function] `vector-clocks/vector_clock.py:VersionedKVStore._receive_replica` — This is where causality-aware replication happens; designing tombstone support here requires handling the case where a tombstone and a live write are concurrent siblings
- [general] `tombstone-gc-and-repair-window` — Investigate how real systems (Cassandra's `gc_grace_seconds`, Riak's delete mode) bound tombstone lifetime relative to maximum expected node downtime
- [function] `read-repair/read_repair.py:get` — Compare this read-repair implementation with dynamo.py's; both have the same resurrection bug, but this one's repair stats tracking would need to distinguish "repaired with value" from "repaired with tombstone"

## Beliefs

- `dynamo-no-delete-method` — Neither `ReplicaNode` nor `DynamoCluster` expose a delete operation; the only mutation path is `put()` which always stores a non-tombstone value
- `read-repair-resurrects-deleted-keys` — `DynamoCluster.get()` read repair at lines 150-160 will overwrite a node that has no entry for a key with the value from any node that still holds it, causing naive deletes to be undone
- `anti-entropy-unions-all-keys` — `anti_entropy_repair()` collects keys from all available nodes via set union (line 183-185), so a key present on any single node will be propagated to all nodes regardless of whether it was intentionally removed elsewhere
- `versioned-value-has-no-tombstone-flag` — The `VersionedValue` dataclass at dynamo.py:14-18 carries only `value`, `version`, and `node_id` with no field to distinguish a live value from a deletion marker

