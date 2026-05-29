# Topic: Study LevelDB's `VersionSet` and `VersionEdit` to see how production systems track which SSTables exist at which levels with key ranges

**Date:** 2026-05-29
**Time:** 09:01

# LevelDB's VersionSet/VersionEdit Pattern — and How This Codebase Approximates It

## The Production Problem

In LevelDB, the core challenge is: **which SSTable files currently exist, at which levels, covering which key ranges?** This is metadata that changes every time a compaction runs (files are created and deleted) or a memtable flushes (a new L0 file appears). LevelDB solves this with three concepts:

1. **`FileMetaData`** — per-file record: file number, file size, smallest key, largest key
2. **`VersionEdit`** — a delta describing what changed: "add file X to level 2, delete file Y from level 1"
3. **`VersionSet`** — the full current state (which files at which levels), reconstructed by replaying a log of `VersionEdit` records from a MANIFEST file

This is essentially **event sourcing for file metadata** — the MANIFEST is an append-only log of edits, and the current `Version` is the materialized view.

## What This Codebase Implements (and What It Doesn't)

### SSTableMetadata — the FileMetaData analog

The `sstable-and-compaction/sstable.py` file defines `SSTableMetadata` (line 34–40):

```python
@dataclass
class SSTableMetadata:
    filepath: str
    min_key: str
    max_key: str
    entry_count: int
    file_size: int
    level: int
    timestamp: float
```

This is the closest analog to LevelDB's `FileMetaData`. It tracks the key range (`min_key`, `max_key`) and which level the file belongs to. The `SSTableWriter.finish()` method (line 89–105) produces this metadata when an SSTable is finalized — note that `level` is hardcoded to `0` at creation time (line 104), which matches LevelDB's behavior where freshly flushed files always land at L0.

The `SSTableReader` reconstructs this metadata on open by reading the first and last entries to determine `min_key` and `max_key` (lines 142–165). This is a simplification — LevelDB stores these in the MANIFEST rather than re-scanning files.

### CompactionManager — where level tracking lives

The test file (`sstable-and-compaction/test_sstable.py`, lines 108–122) shows leveled compaction in action:

```python
mgr = CompactionManager(lcs_dir, strategy='leveled', l0_compaction_trigger=2)
# ... add 3 L0 SSTables ...
result = mgr.run_compaction()
assert result[0].level == 1  # compacted output promoted to L1
```

The `CompactionManager` maintains the current set of SSTables and their levels. When compaction runs, it merges overlapping files and promotes the result to the next level. This is the **operational equivalent** of a `VersionEdit` that says "delete these L0 files, add this new L1 file" — but it's done imperatively rather than through a logged edit.

### The LSM Tree — simpler, no levels at all

The `log-structured-merge-tree/lsm.py` implementation takes a simpler approach. Its `LSMTree` class (line 200+) maintains a flat list of SSTables (`self._sstables`) with no level concept. Compaction (line 319) merges **all** SSTables into one, rather than doing level-by-level promotion:

- `compaction_threshold` (line 204, 208) triggers compaction when the SSTable count gets too high
- The threshold check happens at line 316: `if len(self._sstables) >= self._compaction_threshold`
- The `compact()` method (line 319) does a single all-to-one merge

This is closer to a **size-tiered** strategy with a single tier — there's no `VersionSet` tracking which files belong to which level because there are no levels.

## What's Missing Relative to LevelDB

The key gap is **durability of the version state**. In LevelDB:

1. A `VersionEdit` is written to the MANIFEST *before* the compaction output is visible — this is crash-safe
2. On recovery, the `VersionSet` replays the MANIFEST to reconstruct which files exist at which levels
3. Old `Version` objects are kept alive via reference counting so concurrent readers see a consistent snapshot (MVCC for file metadata)

Neither implementation here has:

- **A MANIFEST / version log** — file metadata is reconstructed by scanning the directory on startup
- **Atomic version transitions** — no mechanism to say "these files were added and these were removed" as a single atomic operation
- **Snapshot isolation for readers** — no reference-counted `Version` objects that let in-progress reads survive a compaction

The `sstable-and-compaction` module has the right data structures (`SSTableMetadata` with `level`, `min_key`, `max_key`) but manages them in-memory only. The `lsm.py` module doesn't even track levels, using sequence numbers (`seq` in the `SSTable` class, line 74) for ordering instead.

## The Design Insight

LevelDB's `VersionSet`/`VersionEdit` pattern exists because SSTable management has the same problem as any database: you need **atomic, durable state transitions** for metadata. The MANIFEST is essentially a write-ahead log for the file catalog. This codebase correctly implements the data-plane side (SSTable format, sparse indexes, merge logic) but simplifies away the metadata-plane — which is fine for learning the algorithms, but is exactly the gap you'd need to close for a production system.

---

## Topics to Explore

- [function] `sstable-and-compaction/sstable.py:CompactionManager` — How the compaction manager decides which SSTables to compact and how it promotes files between levels; this is the closest analog to VersionEdit application
- [function] `log-structured-merge-tree/lsm.py:compact` — The all-to-one merge strategy and how it contrasts with LevelDB's level-by-level compaction; understanding what you lose without level tracking
- [general] `manifest-and-crash-recovery` — How LevelDB's MANIFEST file provides crash-safe version transitions; compare with the WAL crash recovery in `lsm.py` (lines 88–96) which only covers data, not metadata
- [function] `sstable-and-compaction/sstable.py:merge_sstables` — The multi-way merge implementation that underpins compaction; this is the primitive that both size-tiered and leveled strategies use
- [general] `mvcc-version-snapshots` — How LevelDB keeps old Version objects alive for concurrent readers via reference counting, and why this matters for read consistency during compaction

## Beliefs

- `sstable-metadata-tracks-key-range-and-level` — `SSTableMetadata` stores `min_key`, `max_key`, and `level` per file (sstable.py:34–40), providing the per-file key range tracking that LevelDB's `FileMetaData` serves
- `lsm-tree-has-no-level-concept` — The `LSMTree` in lsm.py maintains a flat list of SSTables ordered by sequence number with no level assignment; compaction merges all files into one rather than promoting between levels
- `fresh-sstables-always-start-at-level-zero` — `SSTableWriter.finish()` hardcodes `level=0` (sstable.py:104), matching LevelDB's invariant that flushed memtables always produce L0 files
- `no-manifest-or-durable-version-tracking` — Neither implementation persists SSTable metadata to a manifest file; file catalog state exists only in memory and must be reconstructed from the filesystem on restart
- `leveled-compaction-promotes-to-next-level` — The CompactionManager's leveled strategy produces output SSTables at level N+1, as verified by the test assertion `result[0].level == 1` (test_sstable.py:119)

