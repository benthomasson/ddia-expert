# Topic: How LevelDB's MANIFEST and RocksDB's version sets solve exactly this problem with atomic state transitions

**Date:** 2026-05-28
**Time:** 19:04

# How LevelDB's MANIFEST and RocksDB's Version Sets Solve Atomic State Transitions

## The Problem This Codebase Exposes

The grep for `manifest|MANIFEST|version_set|VersionSet|version_edit|VersionEdit` returned **zero matches**. This is the most important observation — the codebase has a working LSM tree with compaction, but it's missing the mechanism that makes compaction crash-safe.

Look at the compaction flow in `log-structured-merge-tree/lsm.py`. The `compact()` method at line 319 does a k-way merge of all SSTables into a single new SSTable (line 347), then presumably swaps `self._sstables` in memory and deletes the old files. This involves multiple steps that are **not atomic**:

1. Write the new merged SSTable to disk
2. Update the in-memory list (`self._sstables`) to point to the new file
3. Delete the old SSTable files

If the process crashes between step 1 and step 3, you have orphaned files on disk with no record of which ones are "current." If it crashes between step 2 and step 3, the in-memory state is lost (it was never persisted), and on recovery you'd see both old and new files with no way to distinguish them.

The same problem appears during flush. After `_flush()` writes a new SSTable (around line 315), the system must atomically transition from "these N SSTables are current" to "these N+1 SSTables are current." The WAL (line 17–66) protects individual key-value writes, but **nothing protects the SSTable-level metadata**.

## What a MANIFEST Solves

LevelDB's MANIFEST is a write-ahead log for metadata — not for user data, but for *which files constitute the database*. Each state transition is recorded as a **VersionEdit**: a compact record that says "add file X at level L, remove files Y and Z." The MANIFEST file is append-only, and each VersionEdit is written atomically (with checksums, just like the WAL at `write-ahead-log/wal.py` lines 30–36).

The key insight: **the set of live SSTable files is itself mutable state that needs crash protection**, exactly like user data needs the WAL.

Here's how it maps to the missing piece in this codebase:

| This codebase does | MANIFEST would add |
|---|---|
| WAL protects memtable writes (`lsm.py:17–66`) | MANIFEST protects SSTable file set |
| `self._sstables` list tracks live files in memory | VersionEdit log tracks live files on disk |
| `compact()` writes new file + deletes old ones non-atomically | VersionEdit atomically records "add new, remove old" |
| Crash recovery replays WAL only (`lsm.py` — WAL replay) | Recovery replays MANIFEST to reconstruct file set |

## How Version Sets Make This Atomic

RocksDB extends this with **VersionSet** — an in-memory structure where each **Version** is an immutable snapshot of "which SSTables exist at which levels." A compaction doesn't mutate the current version; it creates a *new* Version by applying a VersionEdit. The transition looks like:

```
Version_n  +  VersionEdit  →  Version_{n+1}
(old files)   (add/remove)    (new files)
```

This is an **immutable data structure with pointer swap** — the same pattern you'd use for lock-free concurrent reads. Ongoing reads can continue using Version_n while compaction installs Version_{n+1}. The old version is reference-counted and freed when the last reader finishes.

Compare this to the codebase's `compact()` at line 319: it mutates `self._sstables` in place. Any concurrent reader iterating over that list during compaction would see inconsistent state. A VersionSet eliminates this entirely — readers hold a reference to an immutable version, and the swap to a new version is a single pointer update.

## The CURRENT File: Bootstrapping Recovery

One subtle detail: the MANIFEST itself can be rotated (it grows over time). LevelDB uses a tiny file called `CURRENT` that contains just the filename of the active MANIFEST. On startup:

1. Read `CURRENT` to find the MANIFEST filename
2. Replay the MANIFEST from the beginning to reconstruct the VersionSet
3. Now you know exactly which SSTable files are live

This codebase has no equivalent. The `LSMTree.__init__` (around line 204) would need to scan the directory and guess which `.sst` files are valid — a fundamentally unsafe operation after a crash mid-compaction.

## What's Missing From the Observations

The observations confirm the absence but can't show the solution in action. To fully understand MANIFEST/VersionSet, you'd need:

- The actual LevelDB or RocksDB source (C++), specifically `version_set.cc`, `version_edit.cc`, and the MANIFEST write path
- An implementation of level-based (not just size-tiered) compaction, where tracking per-level file sets makes the Version abstraction essential
- The reference-counting mechanism that lets old Versions stay alive for in-progress iterators

The `sstable-and-compaction/sstable.py` file has `SSTableMetadata` with a `level` field (line 39), suggesting level-aware compaction was intended, but the metadata tracking is in-memory only — exactly the gap a MANIFEST would fill.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:compact` — Trace the non-atomic file swap to understand exactly where a crash would leave inconsistent state
- [file] `sstable-and-compaction/sstable.py` — The `SSTableMetadata` dataclass (line 35) models per-file metadata that a VersionEdit would persist; compare what's tracked vs. what's missing
- [general] `leveldb-version-edit-format` — Study LevelDB's `VersionEdit::EncodeTo` to see the compact binary format for "add file / remove file" records
- [function] `write-ahead-log/wal.py:append_batch` — The batch-with-COMMIT pattern (line 156) is the same atomicity primitive a MANIFEST uses; understanding it here transfers directly
- [general] `mvcc-version-lifetime` — How RocksDB's `SuperVersion` reference-counts live Versions so iterators see a consistent snapshot even during compaction

## Beliefs

- `lsm-no-manifest` — The LSM tree implementation in `lsm.py` has no persistent metadata log; the set of live SSTable files exists only in memory (`self._sstables`) and is not crash-recoverable independently of directory scanning
- `compaction-not-atomic` — `LSMTree.compact()` performs a multi-step file swap (write new SSTable, update in-memory list, delete old files) with no mechanism to make the transition atomic across crashes
- `wal-protects-data-not-metadata` — The WAL in both `lsm.py` and `write-ahead-log/wal.py` protects user key-value writes but does not record SSTable-level state transitions (flush, compaction, level assignment)
- `sstable-metadata-level-field-unused` — `SSTableMetadata.level` in `sstable-and-compaction/sstable.py:39` exists but has no persistent backing; level assignments would be lost on restart without a MANIFEST equivalent

