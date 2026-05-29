# Topic: When is it safe to remove tombstones? The `remove_tombstones` flag exists but the compaction manager never sets it to `True` — explore what conditions make tombstone removal correct

**Date:** 2026-05-28
**Time:** 19:02

I have enough from the observations to give you a thorough answer. Let me trace through what's here.

---

# When Is It Safe to Remove Tombstones?

## The Core Problem

A tombstone is a "delete marker" — instead of physically removing a key, you write a record that says "this key is deleted." This is necessary in log-structured storage because you can't modify data in place. But tombstones cost space, so eventually you want to remove them. The question is: **when can you safely drop them without causing a deleted key to "come back from the dead"?**

## How Tombstones Work in This Codebase

Across the implementations, tombstones take slightly different forms but serve the same purpose:

- **SSTable module** (`sstable-and-compaction/sstable.py:27`): `value: Optional[str]  # None = tombstone` — a `None` value in an `SSTableEntry` means the key was deleted.
- **LSM Tree** (`log-structured-merge-tree/lsm.py:10`): `TOMBSTONE = b""` — an empty byte string sentinel.
- **Bitcask** (`hash-index-storage/bitcask.py:185`): Appends a tombstone record to the log.
- **Multi-leader replication** (`multi-leader-replication/multi_leader.py:37`): Uses an `is_tombstone` boolean flag in the store tuple.

In every case, a read operation checks for the tombstone and returns "not found" instead of the deleted value.

## The `remove_tombstones` Flag

The `merge_sstables` function at `sstable-and-compaction/sstable.py:251` accepts `remove_tombstones: bool = False`. The logic at line 284 is straightforward:

```python
if remove_tombstones and entry.value is None:
    continue  # skip this entry entirely
```

The default is `False` — tombstones are preserved during merge. Tests explicitly pass `remove_tombstones=True` (`test_sstable.py:41`), but the `CompactionManager` never does.

## Why the CompactionManager Doesn't Set It to `True`

This is the critical design question. The compaction manager (`sstable-and-compaction/sstable.py`) uses `merge_sstables` with the default `remove_tombstones=False` because **it can't know whether removal is safe without additional context.** Here's why:

### The Resurrection Problem

Imagine two SSTables:

| SSTable A (newer) | SSTable B (older) |
|---|---|
| `"foo" → None` (tombstone) | `"foo" → "bar"` |

If you compact **only A** and strip the tombstone, then SSTable B still has `"foo" → "bar"`. A subsequent read will find that entry and return `"bar"` — the key has been resurrected. The delete was lost.

### The Safety Conditions

Tombstone removal is safe **if and only if** the compaction covers **all SSTables that could contain an older version of the key.** Specifically:

1. **Full compaction (all SSTables merged into one):** This is what the LSM Tree does at `lsm.py:319-340`. The `compact()` method merges *all* SSTables and explicitly removes tombstones (line 340: `# Remove tombstones during compaction`). This is safe because no older SSTable survives that could contain a superseded live value.

2. **Lowest-level compaction in leveled compaction:** In a leveled scheme, a compaction that covers the *last level* (the bottommost level of the tree) can safely remove tombstones because there is no deeper level where an older value might hide.

3. **All SSTables containing the key are included in the compaction:** More precisely, a tombstone for key K can be removed if every SSTable that *could* contain an entry for K is being included in this compaction run. In size-tiered compaction, this is hard to guarantee for a partial compaction, which is why the flag stays `False`.

### Contrast: LSM Tree vs. SSTable Module

The LSM Tree implementation (`lsm.py:319-340`) gets away with always removing tombstones because its `compact()` method is a **full merge** — it replaces *every* SSTable with a single new one. There are no survivors to cause resurrection.

The SSTable module's `CompactionManager` is more general — it supports both size-tiered and leveled strategies with partial compactions. A size-tiered compaction might only merge a subset of SSTables at a given size tier. A leveled compaction might merge L0 into L1 while L2 still exists. In both cases, blindly removing tombstones would risk resurrection from the SSTables not included in the merge.

### What Would a Correct Implementation Look Like?

The `CompactionManager.run_compaction()` should set `remove_tombstones=True` when:

1. **Size-tiered strategy:** Only during a "major compaction" that merges all SSTables across all tiers into one.
2. **Leveled strategy:** When compacting into the bottommost level, *and* the compaction includes all SSTables in that level whose key ranges overlap with the input.

The current code is **conservatively correct** — it never removes tombstones, so it never risks resurrection. The cost is unbounded tombstone accumulation over time.

### The Replication Dimension

The multi-leader replication module (`multi-leader-replication/multi_leader.py:75-84`) adds another constraint: a tombstone can't be removed until **all replicas have received it**. If you remove a tombstone on node A before node B has seen the delete, node B's replication of the old live value could re-insert the key on node A. This is why distributed systems often use **tombstone TTLs** (e.g., Cassandra's `gc_grace_seconds`) — you keep the tombstone for long enough that all replicas are guaranteed to have received it.

---

## Topics to Explore

- [function] `sstable-and-compaction/sstable.py:merge_sstables` — Trace the full merge logic including timestamp-based conflict resolution to understand how "newer wins" interacts with tombstone propagation
- [function] `log-structured-merge-tree/lsm.py:compact` — The only place tombstones are actually removed — compare its full-merge approach with the CompactionManager's partial-merge approach
- [file] `hash-index-storage/bitcask.py` — Bitcask's `merge()` at line 195 removes tombstones freely because it rebuilds the entire keydir from one merged file — another example of full compaction making removal safe
- [general] `leveled-compaction-tombstone-policy` — How production systems like LevelDB/RocksDB track "bottommost level" to decide when tombstone removal is safe during leveled compaction
- [file] `multi-leader-replication/multi_leader.py` — The replication-aware tombstone lifecycle: when distributed systems need tombstone TTLs beyond just local compaction safety

## Beliefs

- `tombstone-removal-requires-full-key-coverage` — A tombstone for key K can only be safely removed during compaction if every SSTable that could contain an older entry for K is included in that compaction run; otherwise the deleted key can be resurrected from a surviving SSTable
- `compaction-manager-never-removes-tombstones` — The `CompactionManager` in `sstable-and-compaction/sstable.py` always calls `merge_sstables` with the default `remove_tombstones=False`, making it conservatively correct but causing unbounded tombstone accumulation
- `lsm-compact-removes-tombstones-safely` — The LSM Tree's `compact()` method (`lsm.py:319`) removes tombstones because it performs a full merge of all SSTables, guaranteeing no surviving SSTable can contain a superseded live value
- `distributed-tombstone-removal-needs-replication-convergence` — In replicated systems like the multi-leader module (`multi_leader.py`), tombstones cannot be safely removed until all replicas have received the delete, adding a replication-convergence constraint beyond the local compaction-coverage constraint

