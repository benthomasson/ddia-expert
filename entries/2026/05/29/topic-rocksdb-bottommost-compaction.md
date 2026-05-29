# Topic: RocksDB's `BottommostLevelCompaction` enum and `CompactionJob::is_bottommost_level` flag show the production implementation of this concept

**Date:** 2026-05-29
**Time:** 09:02

# Bottommost-Level Compaction: From Reference Implementation to Production

## The Concept in This Codebase

**The observations are insufficient to directly explain RocksDB's `BottommostLevelCompaction` enum or `CompactionJob::is_bottommost_level` flag** — this codebase contains no RocksDB code, and the grep for `bottom` returned zero relevant hits. However, the reference implementations here demonstrate the *underlying problem* that those RocksDB constructs solve.

### Where the Concept Surfaces Implicitly

The key behavior lives in the compaction logic across two files:

**`log-structured-merge-tree/lsm.py:340`** — During compaction, tombstones are removed unconditionally:

```python
# Remove tombstones during compaction
```

This is the simple case: the LSM tree has a flat list of SSTables (`self._sstables`), and `compact()` merges *all* of them into one. Since there's nothing beneath the merged output, it's always safe to drop tombstones. Every compaction is implicitly a "bottommost level" compaction.

**`sstable-and-compaction/sstable.py`** — The `CompactionManager` introduces *leveled compaction* (line 1, lines 108–120 in tests), where SSTables are organized into levels. The test at `test_sstable.py:117` shows `run_compaction()` promoting L0 SSTables to level 1 (`result[0].level == 1`). The `merge_sstables` function at `test_sstable.py:43` accepts a `remove_tombstones=True` parameter — an explicit choice about whether tombstones survive the merge.

### Why This Matters: The Problem RocksDB's Flags Solve

In a leveled compaction scheme, you **cannot** safely remove tombstones during compaction *unless* you're compacting at the bottommost level. If you drop a tombstone at level 1, an older copy of that key might still exist at level 2 — and with the tombstone gone, the deleted key silently reappears.

This reference implementation sidesteps the problem in two ways:

1. **`lsm.py`** has no levels — `compact()` (triggered at `lsm.py:316` when `len(self._sstables) >= self._compaction_threshold`) merges everything, so tombstone removal is always safe.
2. **`sstable.py`** exposes `remove_tombstones` as a caller-controlled flag rather than deriving it from level metadata. The test at `test_sstable.py:43` passes `remove_tombstones=True` explicitly during merge — there's no automatic reasoning about whether deeper levels might hold stale data.

In RocksDB, `BottommostLevelCompaction` is an enum that controls *when* the engine bothers compacting the deepest level (since it's expensive and only needed for space reclamation), and `CompactionJob::is_bottommost_level` is a runtime flag that tells the compaction job "you're at the bottom — it's safe to drop tombstones and perform other cleanup." These two constructs automate what this codebase handles manually or avoids entirely.

### The Gap

The missing piece in these implementations is **level-aware tombstone safety**. Neither implementation tracks whether a compaction output sits above other data that might contradict tombstone removal. A production system like RocksDB must answer: "Does any level below me contain keys that overlap with this compaction's key range?" If yes, tombstones must be preserved. The `is_bottommost_level` flag encodes exactly that answer.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:compact` — The full compaction method that merges all SSTables and unconditionally removes tombstones — trace why this is safe given the flat structure
- [function] `sstable-and-compaction/sstable.py:merge_sstables` — The `remove_tombstones` parameter is the manual equivalent of `is_bottommost_level` — examine how it's used across callers
- [general] `leveled-compaction-tombstone-safety` — Read RocksDB's `CompactionJob::SetupBottomMostLevel()` to see how it determines whether tombstones can be dropped based on overlapping key ranges in lower levels
- [file] `sstable-and-compaction/sstable.py` — The `CompactionManager` class with its `strategy='leveled'` mode — examine how level promotion works and whether level metadata is tracked on SSTable readers
- [general] `rocksdb-bottommost-level-enum` — RocksDB's `BottommostLevelCompaction` enum values (`kSkip`, `kIfHaveCompactionFilter`, `kForce`, `kForceOptimized`) control when to trigger expensive bottom-level compaction for space reclamation

---

## Beliefs

- `lsm-flat-compaction-always-bottommost` — `LSMTree.compact()` in `lsm.py` merges all SSTables into one without level hierarchy, making every compaction implicitly a bottommost-level operation where tombstone removal is always safe
- `sstable-merge-tombstone-flag-is-caller-controlled` — `merge_sstables` in `sstable.py` takes an explicit `remove_tombstones` boolean rather than deriving tombstone safety from level metadata, shifting correctness responsibility to the caller
- `leveled-compaction-promotes-to-level-1` — The leveled compaction strategy in `CompactionManager` promotes L0 SSTables to level 1 after compaction (`result[0].level == 1` in test), but the codebase does not implement deeper level hierarchies or cross-level overlap checks
- `no-bottommost-level-detection-exists` — Neither compaction implementation contains logic to determine whether a compaction output is the bottommost level — the concept that RocksDB's `is_bottommost_level` flag encodes is absent from this codebase

