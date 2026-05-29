# Topic: How RocksDB's `SuperVersion` reference-counts live Versions so iterators see a consistent snapshot even during compaction

**Date:** 2026-05-29
**Time:** 07:26

# SuperVersion Reference Counting for Consistent Snapshots During Compaction

## The Short Answer: This Codebase Doesn't Implement It

The grep results are clear — there are **zero matches** for `SuperVersion`, `super_version`, `ref_count`, or `refcount` anywhere in the repository. This codebase implements the building blocks that motivate SuperVersion (LSM trees with compaction, MVCC snapshots), but stops short of combining them with reference-counted version management.

## What the Codebase *Does* Implement

### LSM Tree Compaction Without Snapshot Safety

In `log-structured-merge-tree/lsm.py:319`, the `compact()` method merges all SSTables into one and removes tombstones (line 340). The problem: it directly mutates `self._sstables` — the list of live SSTables. If an iterator were scanning those SSTables mid-compaction, it would see the rug pulled out from under it. There's no mechanism to keep old SSTables alive while iterators still reference them.

The range scan at line 274 collects iterators over the current set of SSTables, but nothing prevents `compact()` from deleting those files while the scan is in progress.

### MVCC Snapshot Isolation — The Read-Side Analog

`snapshot-isolation/mvcc_database.py` shows the *logical* version of the same problem. Each `Transaction` (line 22) captures `active_at_start` (line 30) — the set of transaction IDs that were in-flight when the transaction began. The `_is_visible()` method (line 74) uses this snapshot to decide which `Version` objects (line 11) a transaction can see.

This is the read-consistency guarantee that SuperVersion provides at the storage layer: a reader should see a frozen point-in-time view even as writers create new versions.

## What SuperVersion Would Add

In RocksDB, a `SuperVersion` bundles three things into one reference-counted object:

1. **The current MemTable** (analogous to the in-memory `SortedDict` used in the LSM tree)
2. **The set of immutable MemTables** awaiting flush
3. **The current `Version`** — the specific set of SSTable files at each level

When an iterator is created, it increments the SuperVersion's ref count. Compaction can install a *new* SuperVersion (pointing to the newly-compacted SSTables), but the old SuperVersion stays alive — and its SSTables stay on disk — until the last iterator releases its reference.

### What's Missing From This Codebase

- **No ref-counting on SSTables or the SSTable list.** `LSMTree.compact()` (line 319) deletes old files immediately with `os.remove()` (visible from the compaction logic). An active `scan()` iterator would crash.
- **No immutable snapshot of the SSTable set.** The `_sstables` list is mutated in place. RocksDB's `Version` is an immutable object — once created, its file list never changes.
- **No separation between "installing a new version" and "cleaning up the old one."** RocksDB decouples these: installation is instant (swap an atomic pointer), cleanup waits for ref count to hit zero.

The MVCC database in `snapshot-isolation/mvcc_database.py` solves the logical equivalent — `_is_visible()` filters versions without deleting old ones — but the LSM tree doesn't apply this pattern to its physical file management.

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:compact` — Trace what happens to `_sstables` during compaction and identify the race window where an active scan would break
- [function] `snapshot-isolation/mvcc_database.py:_is_visible` — Understand the visibility rules that a SuperVersion implicitly enforces at the storage layer
- [file] `sstable-and-compaction/sstable.py` — The `SSTableReader` and compaction strategies here are closer to production design; check whether they protect against mid-compaction reads
- [general] `superversion-refcount-implementation` — Implementing a `SuperVersion` wrapper around the LSM tree's `_sstables` list with `__enter__`/`__exit__` ref counting would be a high-value addition
- [function] `write-skew-detection/ssi_database.py:_snapshot` — Another snapshot mechanism (line 75) that builds a full point-in-time view; compare with how SuperVersion achieves this without copying

## Beliefs

- `lsm-compact-no-snapshot-safety` — `LSMTree.compact()` mutates `_sstables` in place and deletes old SSTable files immediately, with no mechanism to keep them alive for in-progress iterators
- `mvcc-visibility-is-logical-superversion` — `MVCCDatabase._is_visible()` provides the same read-consistency guarantee as SuperVersion but at the logical/transaction layer rather than the physical storage layer
- `no-refcount-in-codebase` — No file in the repository implements reference counting on SSTable files, Version objects, or any storage-layer snapshot structure
- `lsm-scan-race-window` — A range scan started before `compact()` will hold stale Python generator references to SSTable files that `compact()` may delete, creating a file-not-found race condition

