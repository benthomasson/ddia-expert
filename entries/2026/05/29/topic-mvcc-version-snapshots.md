# Topic: How LevelDB keeps old Version objects alive for concurrent readers via reference counting, and why this matters for read consistency during compaction

**Date:** 2026-05-29
**Time:** 13:45

## LevelDB Version Reference Counting — What the Codebase Shows (and Doesn't)

**The short answer: this codebase does not implement LevelDB's Version reference counting.** The observations are insufficient to explain that mechanism from code. Here's what's present, what's missing, and why it matters.

### What the codebase has

The LSM tree implementation in `log-structured-merge-tree/lsm.py` has a working compaction path. At line 319, `compact()` merges all SSTables into one, and at line 316-317, compaction triggers automatically when the SSTable count exceeds a threshold:

```python
if len(self._sstables) >= self._compaction_threshold:
    self.compact()
```

The `SSTable` class (line 69 of `lsm.py`) reads from files on disk with a sparse index, and compaction produces a new merged SSTable. But there is **no concurrency control** around the SSTable list. When `compact()` runs, it mutates `self._sstables` directly — any concurrent reader iterating that same list would see inconsistent state.

### What's missing: the Version pattern

In real LevelDB, this problem is solved by a `Version` object — an immutable snapshot of "which SSTable files exist at each level." The key pieces absent from this codebase:

1. **No `Version` class** — The grep for `class.*Version` found only `VersionedValue` (dynamo.py:13, vector_clock.py:82) and `Version` (mvcc_database.py:9), none of which represent an SSTable manifest.

2. **No reference counting** — The grep for `ref_count|refcount|_refs|retain|release` found only lock-related `release()` methods (fencing_tokens.py:38, lamport.py:153), not lifecycle management for read snapshots.

3. **No `VersionSet` or `current_` pointer** — The grep for `current_version|snapshot|_version` found MVCC snapshots (ssi_database.py:75) and dynamo versioning (dynamo.py:82), but nothing that tracks "which set of SSTables is the current view."

### How it would work (the LevelDB design)

In LevelDB's design, the pattern works like this:

- A **Version** is an immutable list of SSTable files per level. It has a `ref_count`.
- A **VersionSet** holds a `current_` pointer to the latest Version.
- When a reader starts a `Get()` or iterator, it calls `Ref()` on the current Version, incrementing the count.
- When the reader finishes, it calls `Unref()`, decrementing. If the count hits zero, the Version (and its SSTable files) can be deleted.
- When compaction finishes, it creates a **new** Version with updated file lists and installs it as `current_`. The old Version stays alive as long as any reader holds a reference.

This is why it matters for read consistency: a reader that started before compaction continues to see the old set of SSTables — files aren't deleted out from under it. Without this, you'd get one of two problems:

1. **File-not-found errors** — compaction deletes an SSTable that a concurrent reader is scanning
2. **Inconsistent reads** — a reader sees a mix of pre- and post-compaction state

The current `lsm.py` implementation at line 319 simply does `self._sstables = [merged]`, which is fine for single-threaded use (as the tests at `test_lsm.py` demonstrate) but would break under concurrency.

### The MVCC analogy already in the codebase

Interestingly, `snapshot-isolation/mvcc_database.py:9` has a `Version` class, and `write-skew-detection/ssi_database.py:59-67` implements multi-version visibility rules — the same *idea* (readers see a consistent point-in-time view) applied at the row level rather than the file level. LevelDB's Version refcounting is MVCC for the storage engine's metadata.

---

## Topics to Explore

- [file] `snapshot-isolation/mvcc_database.py` — Contains a `Version` class that implements row-level MVCC, the same consistency principle that LevelDB applies to SSTable file sets
- [function] `log-structured-merge-tree/lsm.py:compact` — The single-threaded compaction implementation; understanding what it mutates reveals exactly where reference counting would be needed
- [function] `write-skew-detection/ssi_database.py:_visible_value` — Shows how snapshot-based visibility works at the data level, analogous to how Version objects provide visibility at the file level
- [general] `leveldb-version-set` — Study LevelDB's `version_set.cc` upstream to see `Ref()`/`Unref()` in action; this codebase would need a similar pattern to support concurrent readers during compaction
- [file] `sstable-and-compaction/sstable.py` — The more detailed SSTable implementation with leveled compaction metadata (`SSTableMetadata.level`), which would be the natural place to add Version tracking

## Beliefs

- `lsm-no-concurrency-control` — The LSM tree in `lsm.py` has no locking, reference counting, or Version snapshots; `compact()` mutates `self._sstables` in place, making it unsafe for concurrent readers
- `lsm-compaction-replaces-all` — `LSMTree.compact()` merges all SSTables into a single new SSTable and deletes the originals, rather than doing incremental level-by-level compaction
- `mvcc-pattern-exists-at-row-level` — The codebase implements MVCC visibility (`ssi_database.py:_visible_value`) at the row/key level but not at the SSTable-file level where LevelDB uses Version reference counting
- `sstable-metadata-has-level-field` — `SSTableMetadata` in `sstable.py` tracks a `level` field, indicating the compaction module is structured for leveled compaction even though `lsm.py` does flat merges

