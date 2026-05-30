# Topic: Investigate how real systems (Cassandra's `gc_grace_seconds`, Riak's delete mode) bound tombstone lifetime relative to maximum expected node downtime

**Date:** 2026-05-29
**Time:** 13:38

Now I have a complete picture. Here's the explanation:

---

# Tombstone Lifetime Bounding: How Real Systems Relate to These Reference Implementations

## The Core Problem

In distributed storage, you can't simply delete a key — you must write a **tombstone** (a "this key is dead" marker) so that replicas know the deletion was intentional. Without it, a replica that was down during the delete will still hold the live value, and read repair or anti-entropy will **resurrect the deleted data** onto the nodes that processed the delete.

The question then becomes: how long do you keep the tombstone? Keep it forever and you leak space unboundedly. Remove it too early and a replica that was offline longer than your tombstone lifetime comes back and resurrects everything.

## What the Reference Implementations Do

### Leaderless (Dynamo): No Delete Support At All

The `DynamoCluster` in `leaderless-replication/dynamo.py` has **no `delete()` method** — the only mutation path is `put()` (`topic-leaderless-deletion-gap.md:10`). This isn't an oversight; it's the hardest problem left unsolved in this implementation.

The entry at `topic-leaderless-deletion-gap.md:16-20` walks through exactly why naive deletion (removing from `_store`) is dangerous: `anti_entropy_repair()` collects the **union** of all keys from all available nodes (lines 183-185 of `dynamo.py`) and pushes the highest version everywhere. If even one node still holds the live value, it propagates back to every other node. Read repair (`dynamo.py:139-160`) does the same — it finds the max version and pushes it to stale replicas, treating "no entry" as stale.

The entry explicitly names the design challenge at line 44:

> A tombstone TTL must be longer than the maximum expected node downtime, which is hard to bound.

This is exactly where Cassandra's `gc_grace_seconds` enters the picture.

### Multi-Leader: Tombstones Without GC

`multi-leader-replication/multi_leader.py` does implement tombstones — using a `_TOMBSTONE` sentinel and an `is_tombstone=True` flag in the store tuple (`multi-leader-replication-multi_leader.md:14-17`). Deletes replicate like normal mutations through the Lamport clock + pending queue mechanism.

But there is **no garbage collection**. Tombstones accumulate without bound (invariant 3: "Tombstones are permanent until overwritten," `multi-leader-replication-multi_leader.md:102-103`). The entry at `topic-tombstone-gc-safety.md:79` explains the constraint:

> A tombstone can't be removed until **all replicas have received it**. If you remove a tombstone on node A before node B has seen the delete, node B's replication of the old live value could re-insert the key on node A.

### Local Storage (LSM/SSTable/Bitcask): Compaction-Based Cleanup

On the single-node storage side, the implementations handle tombstone lifetime through compaction:

- **LSM Tree** (`lsm.py:319-340`): Full compaction merges ALL SSTables into one, making tombstone removal trivially safe — no surviving SSTable can contain a superseded value.
- **SSTable/CompactionManager**: The `merge_sstables` function accepts `remove_tombstones: bool = False` (`sstable.py:251`), but the CompactionManager **never sets it to `True`** because partial compactions can't guarantee safety (`topic-tombstone-gc-safety.md:36-40`).
- **Bitcask**: Tombstones are filtered during `compact()`, which rebuilds the entire keydir from merged files — another full-merge approach.

## How Real Systems Bound Tombstone Lifetime

### Cassandra's `gc_grace_seconds`

Referenced but not implemented in these modules. The pattern works like this:

1. When a delete is issued, a tombstone is written with a creation timestamp.
2. The tombstone **must survive** for at least `gc_grace_seconds` (default: 864,000 seconds = 10 days).
3. During that window, repair processes (anti-entropy, read repair) propagate the tombstone to all replicas.
4. After `gc_grace_seconds` expires, SSTable compaction is **allowed** to drop the tombstone.
5. The critical invariant: `gc_grace_seconds` must be **longer than the maximum expected node downtime**. If a node is down for 11 days and your grace period is 10 days, the tombstone gets purged before that node receives it, and the node's stale live value resurrects via repair.

This maps directly onto what the reference implementations are missing. The `topic-leaderless-deletion-gap.md:51` entry draws the analogy explicitly: the TTL-based hint expiration in `hinted-handoff/hinted_handoff.py` is "directly analogous to tombstone GC" — both trade a bounded lifetime against the risk that an offline node outlasts the window.

### Riak's Delete Mode

Riak offers three delete modes that represent different points on the safety-vs-space tradeoff:

- **`keep`**: Tombstones are retained indefinitely (like the multi-leader implementation here). Safe but accumulates dead entries.
- **`timeout`** (default 3 seconds): Tombstones are eligible for reaping after a short TTL. Relies on all replicas being available at delete time — a much weaker guarantee than Cassandra's 10-day grace.
- **`immediate`**: Tombstone is removed as soon as all vnodes acknowledge the delete. This is the most space-efficient but the most dangerous — any node downtime during the delete window causes resurrection.

### The Fundamental Tradeoff

Both Cassandra and Riak are solving the same equation that `topic-tombstone-gc-safety.md:54` identifies:

> Tombstone removal is safe **if and only if** the compaction covers **all SSTables that could contain an older version of the key.**

In a distributed system, "all SSTables" generalizes to "all replicas." The tombstone lifetime must exceed the **maximum window during which a replica could be offline and still return with a stale live value**. The implementations here capture both extremes:

| Approach | Implementation | Analog |
|----------|---------------|--------|
| Never remove | Multi-leader (`multi_leader.py`) | Riak `keep` mode |
| Remove on full merge | LSM Tree (`lsm.py:compact`) | Single-node compaction (no replica concern) |
| Conservative default | SSTable (`sstable.py:merge_sstables`) | Pre-`gc_grace_seconds` — safe but leaks |
| No delete at all | Dynamo (`dynamo.py`) | The problem statement itself |

## What's Missing for a Complete Implementation

To add Cassandra-style tombstone GC to the Dynamo module, you would need:

1. **A `deleted` flag on `VersionedValue`** (`dynamo.py:14-18`) — currently it carries only `value`, `version`, and `node_id` (`topic-leaderless-deletion-gap.md:36`).
2. **A `delete()` method** on both `ReplicaNode` and `DynamoCluster` that writes a tombstone with the current timestamp.
3. **Timestamp tracking on tombstones** to know when they were created.
4. **A `gc_grace_seconds` configuration** that controls how long tombstones survive.
5. **Integration with anti-entropy** so tombstones participate in the "highest version wins" comparison rather than being treated as "key not found."
6. **A GC sweep** that removes tombstones older than `gc_grace_seconds`, callable during compaction or as a background process.

---

## Topics to Explore

- [function] `leaderless-replication/dynamo.py:anti_entropy_repair` — Trace how the key-union + max-version logic would need to change to propagate tombstones instead of resurrecting deleted keys
- [file] `hinted-handoff/hinted_handoff.py` — The TTL-based hint expiration at lines 7-18 is the closest existing pattern to `gc_grace_seconds`; compare the tradeoff between hint TTL and tombstone TTL
- [function] `vector-clocks/vector_clock.py:VersionedKVStore._receive_replica` — Causality-aware replication with dominance checks; a tombstone version must dominate the live value for deletion to propagate, making this the most complex place to add tombstone support
- [general] `merkle-tree-anti-entropy` — Real Cassandra/Dynamo use Merkle trees to efficiently detect replica divergence instead of full-keyspace scans; understanding this is prerequisite to implementing efficient tombstone propagation at scale
- [general] `tombstone-compaction-interaction` — How `gc_grace_seconds` interacts with SSTable compaction strategy (size-tiered vs. leveled) to determine when a tombstone is actually physically removed from disk

## Beliefs

- `distributed-tombstone-gc-requires-downtime-bound` — Any tombstone garbage collection strategy must define a maximum tolerated node downtime; tombstones removed before a down node receives them cause data resurrection via read repair or anti-entropy
- `dynamo-anti-entropy-unions-all-keys` — `anti_entropy_repair()` discovers keys via set union across all available nodes (dynamo.py:183-185), so a key present on any single node propagates to all others — making naive deletes impossible without tombstones
- `gc-grace-seconds-is-repair-window` — Cassandra's `gc_grace_seconds` (default 10 days) defines the window during which anti-entropy must propagate a tombstone to all replicas; it is not a storage optimization but a correctness parameter tied to maximum expected node downtime
- `reference-implementations-lack-tombstone-ttl` — None of the DDIA reference implementations combine tombstone support with time-bounded garbage collection; the multi-leader module stores tombstones indefinitely and the Dynamo module has no delete support at all
- `tombstone-lifetime-space-safety-tradeoff` — Longer tombstone lifetimes waste storage but tolerate longer outages; shorter lifetimes save space but risk resurrection — Riak's three delete modes (`keep`/`timeout`/`immediate`) represent three explicit points on this spectrum

