# File: sstable-and-compaction/tester_test_sstable.py

**Date:** 2026-05-29
**Time:** 06:57

I'll work from the test file content provided, which gives a clear picture of the implementation's public API.

---

# `sstable-and-compaction/tester_test_sstable.py`

## Purpose

This is the **verification test suite** for an SSTable (Sorted String Table) and compaction implementation — a reference implementation of the storage engine concepts from DDIA Chapter 3. The `tester_` prefix distinguishes it from `test_sstable.py` (likely the author's own tests), marking this as the external/graded test suite that validates the implementation against a specification.

The file tests the full lifecycle of SSTables: writing sorted key-value entries to disk, reading them back via point lookups and range scans, merging multiple SSTables (k-way merge with timestamp-based conflict resolution), and running compaction strategies (size-tiered and leveled).

## Key Components

### `make_sstable(tmpdir, name, entries, block_size=4)` — Test Helper

A factory that writes a list of `(key, value, timestamp)` tuples into an SSTable file via `SSTableWriter`. Returns `(path, SSTableMetadata)`. The `block_size=4` default creates small blocks to exercise the sparse index — with only 4 entries per block, a 5-entry SSTable spans at least 2 blocks, forcing index lookups rather than linear scans.

### Test Classes

| Class | What it validates |
|-------|-------------------|
| `TestWriteReadRoundTrip` | Write entries → read them back via `scan()` and `get()`. The core serialization contract. |
| `TestRangeScan` | `range_scan(start, end)` with exclusive end boundary, plus empty-result case. |
| `TestTombstones` | Deletion markers: writing `None` as a value, reading it back as a tombstone entry. |
| `TestMerge` | `merge_sstables()` — 2-way merge with timestamp conflict resolution, tombstone removal during compaction, and 5-way merge correctness. |
| `TestCompaction` | `CompactionManager` with both `size_tiered` and `leveled` strategies — triggering, execution, and post-compaction state. |
| `TestEdgeCases` | Empty SSTable, single-entry SSTable, and merge where all entries are tombstones. |

## Patterns

**Pytest fixtures over setUp/tearDown.** Every test method takes `tmp_path` (a pytest built-in fixture), ensuring isolated temp directories with automatic cleanup. No shared mutable state between tests.

**Helper-driven test setup.** `make_sstable` abstracts the write-path ceremony so tests read declaratively — you see the data, not the plumbing. The entries are always pre-sorted alphabetically, which is an SSTable invariant the helper silently upholds.

**Test numbering in docstrings.** The class docstrings reference test numbers ("Test 1", "Tests 5-7"), suggesting a spec or rubric this suite maps to. This is typical of the `tester_test_*` pattern across the repo — each file validates a numbered checklist.

**Readers-as-merge-inputs.** Tests create `SSTableReader` instances and pass them directly to `merge_sstables()`, confirming the implementation uses a streaming iterator interface rather than loading entire files into memory.

## Dependencies

**Imports from `sstable` module:**
- `SSTableWriter` — builds SSTable files from sorted entries
- `SSTableReader` — reads SSTables via `get()`, `scan()`, `range_scan()`
- `SSTableEntry` — data class for key/value/timestamp tuples (imported but used implicitly via returned objects)
- `SSTableMetadata` — statistics about a written SSTable (entry count, key range)
- `CompactionManager` — orchestrates compaction with pluggable strategies
- `merge_sstables` — standalone k-way merge function
- `MAGIC` — imported but not used directly in tests; likely a file-format magic number

**External:** `os`, `sys`, `tempfile` (tempfile imported but unused — `tmp_path` fixture handles it), `pytest`.

**Nothing imports this file** — it's a leaf test module run by pytest.

## Flow

Each test follows the same pattern:

1. **Write phase:** `make_sstable()` creates an `SSTableWriter`, calls `add(key, value, timestamp)` for each entry in sorted order, and calls `finish()` to flush the file and return metadata.
2. **Read phase:** An `SSTableReader` opens the file, then exercises one of `get()`, `scan()`, or `range_scan()`.
3. **Assert phase:** Checks entry count, key ordering, value correctness, timestamp conflict resolution, or compaction-level assignment.

For merge tests, steps 1-2 repeat for multiple SSTables, then `merge_sstables()` or `CompactionManager.run_compaction()` produces a new merged SSTable that is scanned for correctness.

## Invariants

- **Sorted key order.** All input entries are pre-sorted alphabetically. The SSTable format requires this — the tests never exercise out-of-order insertion (that's the implementation's job to reject or the caller's to prevent).
- **Timestamp wins conflicts.** When the same key appears in multiple SSTables, the entry with the highest timestamp survives merge. Tested explicitly: `apple` at t=2.0 beats `apple` at t=1.0.
- **Range scan is `[start, end)`.** `range_scan("banana", "elderberry")` returns banana, cherry, date — inclusive start, exclusive end. The assert `len(results) == 3` locks this down.
- **Tombstone semantics.** A `None` value is a tombstone. It persists through normal reads (`get` returns an entry with `value is None`), but `merge_sstables(..., remove_tombstones=True)` strips them. Without that flag, tombstones survive merges.
- **Compaction triggers.** Size-tiered compaction fires when SSTable count meets `min_threshold`. Leveled compaction fires when L0 count meets `l0_compaction_trigger`. After leveled compaction, the result is placed at `level == 1`.

## Error Handling

The tests don't exercise error paths — no tests for corrupt files, out-of-order keys, duplicate keys, or disk errors. This is a happy-path validation suite. Point lookup misses (`get("fig")`) return `None` rather than raising, which is tested.

---

## Topics to Explore

- [file] `sstable-and-compaction/sstable.py` — The implementation itself: block format, sparse index structure, and the k-way merge algorithm
- [function] `sstable-and-compaction/sstable.py:merge_sstables` — How the k-way merge resolves conflicts and handles tombstone removal — likely uses a heap/priority queue
- [file] `sstable-and-compaction/test_sstable.py` — The author's own tests, likely covering error paths and internal details that the tester suite skips
- [general] `sstable-block-format` — How `block_size` affects the sparse index and binary search within blocks — the default of 4 in tests is suspiciously small and worth understanding
- [file] `log-structured-merge-tree/lsm.py` — The LSM tree implementation that likely builds on this SSTable layer, adding a memtable and multi-level compaction

## Beliefs

- `sstable-range-scan-exclusive-end` — `SSTableReader.range_scan(start, end)` uses inclusive start and exclusive end boundaries
- `sstable-timestamp-conflict-resolution` — During SSTable merge, the entry with the highest timestamp wins when the same key appears in multiple inputs
- `sstable-tombstone-optional-removal` — Tombstones survive merges by default; they are only stripped when `merge_sstables()` is called with `remove_tombstones=True`
- `sstable-leveled-compaction-promotes-to-l1` — After leveled compaction triggers, the resulting SSTable is assigned to level 1
- `sstable-tester-tests-happy-path-only` — The tester suite validates correct behavior but does not test error conditions like corrupt files or out-of-order insertion

