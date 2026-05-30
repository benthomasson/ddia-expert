# Topic: These are state-based (CvRDTs); delta-state and operation-based (CmRDTs) variants trade different costs — worth understanding the tradeoffs discussed in DDIA

**Date:** 2026-05-29
**Time:** 14:04

# State-Based CRDTs (CvRDTs) in This Implementation

Every CRDT in `conflict-free-replicated-data-types/crdts.py` follows the same pattern: each replica maintains its **full local state**, and synchronization happens by shipping that entire state to peers via a `merge()` method. This is the defining characteristic of **convergent replicated data types (CvRDTs)** — also called state-based CRDTs.

## How State-Based Replication Works Here

Look at `GCounter.merge()` (line 24–27):

```python
def merge(self, other):
    for rid, count in other.counts.items():
        self.counts[rid] = max(self.counts.get(rid, 0), count)
    return self
```

The entire `counts` dictionary is the state. To sync, one replica sends its full `counts` map to another, and the receiver takes the element-wise `max`. This works because `max` forms a **join semilattice** — it's commutative, associative, and idempotent. The tests in `test_crdts.py` lines 30–50 (`TestGCounterProperties`) explicitly verify all three properties.

The same pattern repeats for every type:

- **`PNCounter.merge()`** (line 60–63): delegates to two `GCounter.merge()` calls — one for increments, one for decrements.
- **`LWWRegister.merge()`** (line 105–111): compares `(timestamp, writer_id)` tuples, keeping the higher one. The tuple comparison is the lattice join.
- **`ORSet.merge()`** (line 168–181): unions active tags and tombstone sets, then subtracts tombstones from active tags. The lattice is over sets with union.

Every `merge()` mutates `self` and returns `self`, making it easy to chain. Every type also exposes a `state()` method that serializes the full replica state — this is what would go over the wire in a real deployment.

## What This Approach Costs (and What the Alternatives Trade)

DDIA discusses three CRDT dissemination strategies. Here's how they compare:

| | **State-based (CvRDT)** — this code | **Operation-based (CmRDT)** | **Delta-state** |
|---|---|---|---|
| **What's sent** | Full state on every sync | Individual operations (e.g., "increment by 3") | Only the state *difference* since last sync |
| **Network cost** | O(state size) per sync — `GCounter` sends all replica counts even if only one changed | O(operation size) — just the op | O(delta size) — between the two extremes |
| **Delivery requirements** | None — merge is idempotent, redelivery is fine | Exactly-once, causally ordered delivery | At-least-once with causal metadata |
| **Implementation complexity** | Low — just implement `merge()` | Higher — need reliable causal broadcast middleware | Medium — need delta tracking and anti-entropy |
| **Metadata growth** | Visible here: `ORSet._tombstones` (line 142) grows monotonically and is never compacted | Same problem, but can sometimes be GC'd with causal stability | Can compact deltas after acknowledgment |

The `ORSet` implementation makes the state-based cost concrete. Every `remove()` (line 155–160) moves tags into `_tombstones`, and `merge()` (line 168) unions tombstones from both sides. **Tombstones never shrink.** In a production system with frequent add/remove churn, this set grows without bound. An operation-based OR-Set avoids this because the "remove" operation only needs to reference the specific tags being removed — no persistent tombstone set needed — but it requires the network layer to guarantee causal delivery.

## Why This Implementation Chose State-Based

State-based CRDTs are the right choice for a reference implementation because they're **self-contained**. The `CRDTReplicaGroup` helper (imported in tests, line 6) can simulate replication by just calling `merge()` between in-memory objects — no message broker, no causal delivery layer, no sequence numbers. The tests at lines 83–89 (`TestPNCounterMerge`) show how simple this makes verification: create replicas, mutate them independently, call `sync_all()`, assert convergence.

An operation-based version would need infrastructure (reliable broadcast, duplicate detection, causal ordering) that would obscure the CRDT logic itself. Delta-state would be a reasonable middle ground but adds bookkeeping for tracking what each peer has already seen.

## Topics to Explore

- [function] `conflict-free-replicated-data-types/crdts.py:ORSet.merge` — The most complex merge in this module; understanding how tag unions minus tombstone unions preserve add-wins semantics is key to grasping CvRDT lattice design
- [general] `operation-based-crdt-comparison` — Implement a CmRDT version of GCounter with an operation log and causal delivery layer to feel the infrastructure cost firsthand
- [general] `tombstone-compaction-strategies` — The ORSet tombstone set grows without bound; explore how causal stability or version vectors enable garbage collection in production CRDT systems
- [function] `conflict-free-replicated-data-types/crdts.py:LWWRegister.merge` — The timestamp+replica_id tiebreaker is a design choice with real data-loss implications; compare with multi-value registers that preserve conflicts instead
- [general] `delta-state-crdts` — Delta-state CRDTs (Almeida et al., 2018) solve the bandwidth problem of CvRDTs without requiring CmRDT delivery guarantees — the practical sweet spot for most production systems

## Beliefs

- `crdts-are-state-based-cvrdts` — All four CRDT types (GCounter, PNCounter, LWWRegister, ORSet) use state-based replication via a `merge()` method that implements a join-semilattice; no operation log or causal delivery is used
- `orset-tombstones-grow-monotonically` — `ORSet._tombstones` is only ever unioned during `merge()` and added to during `remove()`; there is no compaction or garbage collection, so the tombstone set grows without bound
- `merge-mutates-self-and-returns-self` — Every CRDT `merge()` method mutates the receiver in place and returns `self`, enabling chaining but meaning callers must `deepcopy()` before merging if the original state must be preserved
- `lww-register-tiebreaks-on-replica-id` — When two `LWWRegister` writes have identical timestamps, the one with the lexicographically higher `replica_id` wins, making conflict resolution deterministic but arbitrary

