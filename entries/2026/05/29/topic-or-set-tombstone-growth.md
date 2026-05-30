# Topic: The tombstone set in `ORSet` grows without bound; production systems (e.g., Riak) use causal context or garbage collection to address this

**Date:** 2026-05-29
**Time:** 14:03

## The ORSet Tombstone Growth Problem

The `ORSet` in `conflict-free-replicated-data-types/crdts.py` tracks removals by accumulating tombstones in `self._tombstones` (line 140), a set that **only ever grows** — nothing in the implementation ever removes entries from it.

### How tombstones accumulate

When you call `remove()` (line 148), every active tag for that element moves into `_tombstones`:

```python
self._tombstones.update(self._elements[element])
del self._elements[element]
```

When you call `merge()` (line 158), the two tombstone sets are unioned:

```python
merged_tombstones = self._tombstones | other._tombstones
```

Every tag that was ever removed stays in `_tombstones` forever. If a replica adds and removes the same element 1,000 times, that's 1,000 tags permanently in the tombstone set — even though only the most recent state matters.

### Why the tombstones can't simply be dropped

The tombstone set is load-bearing during merge. At line 165:

```python
merged_tags = (self_tags | other_tags) - merged_tombstones
```

If replica A removes element "x" and replica B hasn't synced yet, B still has "x"'s tags in its active set. When they merge, A's tombstones are what suppress B's stale tags. Drop the tombstones too early and removed elements reappear — a "zombie resurrection" bug.

### What production systems do instead

This implementation uses the "tag + tombstone" approach, which is correct but naive. Production systems avoid unbounded tombstone growth through two main strategies:

**Causal context (Riak's approach).** Instead of storing individual tombstones, each replica maintains a compact *version vector* (also called a "causal context" or "dot context") that summarizes all tags it has ever seen. A tag `(replica_id, seq)` is implicitly tombstoned if `seq` is at or below the version vector's entry for that replica. This compresses an arbitrary number of tombstones into a single integer per replica. The grep for `VectorClock|version_vector|dot|tag` returned zero matches in the CRDT module — this implementation has no such mechanism.

**Garbage collection with causal stability.** Once all replicas have observed a remove operation (confirmed via anti-entropy or protocol-level acknowledgment), the tombstone is "causally stable" and safe to discard. This requires distributed coordination — you need to know what every replica has seen — which is why the textbook implementation skips it.

### Contrast with the LSM tree

Interestingly, the LSM tree module in this same repo (`log-structured-merge-tree/lsm.py`, line 320) *does* implement tombstone garbage collection: its `compact()` method removes tombstones during compaction. That works because the LSM tree is a single-node data structure — there's no risk of a remote replica holding a stale view that needs the tombstone to prevent resurrection. The distributed setting of CRDTs makes the same optimization fundamentally harder.

## Topics to Explore

- [function] `conflict-free-replicated-data-types/crdts.py:ORSet.merge` — Trace how the tombstone union at line 163 prevents concurrent-add-wins from being defeated, and how it makes tombstones monotonically accumulate
- [file] `log-structured-merge-tree/lsm.py` — Compare single-node tombstone GC (compaction removes tombstones) with the distributed case where GC requires causal stability
- [general] `dotted-version-vectors` — The technique Riak 2.0+ uses to replace per-tag tombstones with a compact causal context, eliminating the growth problem entirely
- [function] `conflict-free-replicated-data-types/crdts.py:ORSet.remove` — Consider what would break if `remove()` only tombstoned a subset of tags (hint: test 13, `TestORSetRemoveKnownTags`)
- [general] `crdt-delta-state` — Delta-state CRDTs transmit only recent changes rather than full state, which also enables scoped tombstone lifetimes

## Beliefs

- `orset-tombstone-set-monotonic` — `ORSet._tombstones` is append-only: tags are added during `remove()` and `merge()` but never deleted, so the set grows without bound over the lifetime of the replica
- `orset-no-causal-context` — The ORSet implementation uses no version vector, dot context, or causal summary; each removed tag is stored individually as a `(replica_id, seq)` tuple in the tombstone set
- `orset-tombstone-required-for-merge-correctness` — Dropping tombstones would cause removed elements to reappear when merging with a replica that still holds the element's active tags (zombie resurrection)
- `lsm-tombstone-gc-not-applicable-to-crdts` — The LSM tree's `compact()` safely removes tombstones because it's single-node; the ORSet cannot do the same without distributed knowledge of what all replicas have observed

