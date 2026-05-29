# Topic: When is it safe to remove tombstones? Relates to DDIA's discussion of compaction and the risk of "resurrecting" deleted data if tombstones are dropped too early

**Date:** 2026-05-29
**Time:** 06:59

# When Is It Safe to Remove Tombstones?

## The Core Problem

DDIA warns that if you drop a tombstone before all copies of the deleted key have been eliminated, the deleted data can "resurrect" — an older SSTable or replica still holding the live value will appear authoritative because no tombstone exists to contradict it. This codebase demonstrates three different approaches to tombstone removal, each with different safety properties — and different risks.

## How Tombstones Work in This Codebase

Across all three storage engines, deletion never physically removes data. Instead, a **tombstone marker** is appended:

| Engine | Tombstone representation | Where defined |
|--------|------------------------|---------------|
| LSM Tree | `TOMBSTONE = b""` (empty bytes) | `log-structured-merge-tree/lsm.py:10` |
| SSTable | `value: None` + `TOMBSTONE_MARKER = 0xFF` byte | `sstable-and-compaction/sstable.py:12,27` |
| Bitcask | Empty string value (`val_size == 0`) | `hash-index-storage/bitcask.py:185` |
| Multi-leader | `is_tombstone: True` flag in tuple | `multi-leader-replication/multi_leader.py:37,84` |

In every case, the tombstone must persist long enough to shadow the deleted key wherever it might still exist.

## Three Compaction Strategies, Three Safety Models

### 1. Bitcask: Safe — Single File, Full Visibility

In `hash-index-storage/bitcask.py:195`, `compact()` merges all **immutable** files (not the active file). Because Bitcask holds the complete `keydir` (an in-memory hash index of every live key), it has perfect knowledge of what's current. During `_scan_data_file` (line ~140), if `val_size == 0` the key is removed from `keydir`:

```python
if val_size == 0:
    # Tombstone - remove from keydir
    self.keydir.pop(key, None)
```

**Why this is safe:** Bitcask's `keydir` is the single source of truth. When compaction rebuilds from all immutable files in order, tombstones naturally cancel out older values. Once the merged file is written, the tombstone has done its job — no other copy of the data exists outside the files being compacted. The key isn't in `keydir`, so it can't be read.

**The risk it avoids:** It only compacts *immutable* files (`fid != self.active_file_id`). If it included the active file, a concurrent write could land between reading and writing, creating a lost update. This isn't a tombstone issue per se, but it's the same class of problem — incomplete visibility leads to data loss.

### 2. LSM Tree: Unsafe in the General Case

In `log-structured-merge-tree/lsm.py:319-340`, `compact()` merges *all* SSTables into one and unconditionally removes tombstones:

```python
# Remove tombstones during compaction  (line 340)
```

The merge at line ~295 already excludes tombstones from the result ("Build result, excluding tombstones"). This is **only safe because the implementation merges ALL SSTables at once** — a full compaction. If this were a partial compaction (merging only some SSTables, as real LSM trees like LevelDB do), removing tombstones would risk resurrection.

**The DDIA scenario that would break:** Imagine SSTables at levels L0, L1, L2. If you compact L0 and L1, dropping tombstones, but L2 still has the old live value — the delete is lost. The key reappears from L2. This is exactly what DDIA warns about, and it's why production LSM trees only remove tombstones when compacting the **bottommost level** (the level with no older data beneath it).

This implementation sidesteps the problem by only supporting "compact everything into one" — the nuclear option. The test at `test_lsm.py:68-83` confirms this: it sets `compaction_threshold=100` to prevent auto-compaction, then manually calls `compact()` which merges everything.

### 3. SSTable Merge: Explicitly Parameterized

The `merge_sstables` function at `sstable-and-compaction/sstable.py:251` takes an explicit `remove_tombstones: bool = False` parameter:

```python
if remove_tombstones and entry.value is None:  # line 284
```

**This is the most honest design.** It pushes the safety decision to the caller. The default is `False` — don't remove tombstones — which is the conservative choice. The caller must opt in to removal, and presumably knows whether all data for those keys is contained in the SSTables being merged.

The test at `sstable-and-compaction/test_sstable.py:41` passes `remove_tombstones=True` explicitly, confirming this is a deliberate choice point.

## The Multi-Leader Problem: Tombstones Across Replicas

The multi-leader replication code (`multi-leader-replication/multi_leader.py:37-116`) reveals a harder variant of the problem. Tombstones carry a timestamp and origin node:

```python
self._store[key] = (actual_val, remote_ts, remote_node, is_tombstone)  # line 116
```

In a replicated system, a tombstone can only be safely removed when **every replica** has received it. If node A deletes key `x` and compacts away the tombstone before node B syncs, node B's stale copy of `x` will propagate back to A on the next replication cycle — classic resurrection.

This implementation doesn't compact tombstones from the multi-leader store at all (there's no compaction method on the `MultiLeaderNode`), which is the safe default. DDIA discusses this as requiring either vector clocks or a garbage collection protocol where nodes agree a tombstone has propagated to all replicas before removal.

## The Leaderless Case: No Tombstones At All

Notably, `leaderless-replication/dynamo.py` has **no tombstone mechanism**. Deletes aren't implemented — the `anti_entropy_repair` method pushes the highest-version value to stale replicas, but there's no concept of a deletion version. If you wanted to add deletes, you'd need to either:
1. Use a tombstone value with a version that wins over live values, or  
2. Use a "deleted" flag analogous to the multi-leader approach

Without this, "delete then repair" would resurrect the data — the exact problem DDIA describes.

## Summary: When Is Removal Safe?

| Condition | Safe to remove? | Why |
|-----------|----------------|-----|
| Full compaction (all data merged) | Yes | No older copy can exist — Bitcask and LSM both do this |
| Partial compaction, bottommost level | Yes | No level below can hold an older live value |
| Partial compaction, non-bottom level | **No** | Older levels may still hold the pre-delete value |
| Replicated system, not all replicas synced | **No** | Unsynced replica will propagate stale value back |
| Replicated system, all replicas confirmed | Yes | All copies of the old value have been superseded |

The codebase demonstrates the first case (full compaction) correctly and avoids the others by either not implementing partial compaction or not implementing tombstone GC in replicated stores.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:compact` — The full-merge compaction that avoids partial-compaction resurrection risk; compare to how LevelDB/RocksDB handle per-level tombstone retention
- [function] `sstable-and-compaction/sstable.py:merge_sstables` — The `remove_tombstones` parameter design pattern; trace how callers decide when to pass `True`
- [file] `multi-leader-replication/multi_leader.py` — Tombstone propagation during `apply_remote_changes` (line 101-116); explore what would break if tombstone GC were added without a quorum protocol
- [general] `leaderless-deletion-gap` — The Dynamo implementation has no delete/tombstone support; designing one that's safe under read repair and anti-entropy would be a valuable exercise
- [function] `hash-index-storage/bitcask.py:_scan_data_file` — How the keydir rebuild handles tombstones during crash recovery; consider what happens if the process crashes mid-compaction

## Beliefs

- `lsm-full-compaction-only` — LSM Tree `compact()` merges all SSTables into one and removes all tombstones; it does not support partial/leveled compaction, which avoids the resurrection bug but sacrifices write amplification control
- `sstable-merge-tombstone-opt-in` — `merge_sstables` defaults to `remove_tombstones=False`, requiring callers to explicitly opt into tombstone removal — the safe default for partial merges
- `bitcask-immutable-only-compaction` — Bitcask `compact()` excludes the active file (`fid != self.active_file_id`), ensuring concurrent writes to the active file cannot be lost during compaction
- `dynamo-no-delete-support` — The leaderless replication implementation has no delete or tombstone mechanism; adding deletes without tombstones would cause resurrection via read repair or anti-entropy
- `multi-leader-no-tombstone-gc` — Multi-leader replication stores tombstones indefinitely with no compaction or garbage collection, which is safe but will accumulate dead entries without bound

