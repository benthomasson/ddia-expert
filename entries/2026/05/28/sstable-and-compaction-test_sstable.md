# File: sstable-and-compaction/test_sstable.py

**Date:** 2026-05-28
**Time:** 19:00

I'll explain based on the test file itself — it reveals the full public API of the SSTable implementation.

---

# `sstable-and-compaction/test_sstable.py`

## Purpose

This is a **smoke test / verification script** for the SSTable (Sorted String Table) and compaction implementation. It's not a pytest-style test suite — it's a top-level script that runs sequentially inside a single `with tempfile.TemporaryDirectory()` block, using bare `assert` statements and printing progress markers. Its job is to exercise every major code path of the `sstable` module and fail fast if anything is broken.

It covers: writing, reading, point lookups, range scans, tombstones, merging (2-way and 5-way), empty/single-entry edge cases, and both size-tiered and leveled compaction strategies.

## Key Components

The test doesn't define classes or functions — it's a linear script. But it exercises these components from `sstable.py`:

| Component | Contract (inferred from usage) |
|---|---|
| **`SSTableWriter(path, block_size)`** | Builds an SSTable file on disk. `add(key, value, timestamp)` appends entries in sorted order. `finish()` returns metadata. `value=None` means tombstone (deletion marker). |
| **`SSTableReader(path)`** | Opens an SSTable for reads. `get(key)` → entry or `None`. `range_scan(start, end)` → iterator of entries in `[start, end)`. `scan()` → full iterator. `metadata()` → metadata object. |
| **`merge_sstables(readers, output_path, remove_tombstones=False)`** | N-way merge of multiple SSTables into one. Deduplicates by key, keeping the entry with the highest timestamp. When `remove_tombstones=True`, drops entries where `value is None`. Returns an `SSTableReader` on the merged file. |
| **`CompactionManager(dir, strategy, ...)`** | Manages when and how to compact SSTables. `add_sstable(reader)` registers a table. `needs_compaction()` → bool. `run_compaction()` → list of new SSTable readers. Supports `'size_tiered'` (threshold-based) and `'leveled'` (L0 trigger-based) strategies. |
| **Metadata object** | Has at least `entry_count`, `min_key`, `max_key`. |
| **Entry object** | Has at least `key`, `value`. |

## Patterns

**Script-as-test**: The entire file is a single imperative script — no test framework, no fixtures, no teardown. The `tempfile.TemporaryDirectory()` context manager handles cleanup. This is the simplest possible verification approach: run it, and if it exits cleanly with "ALL TESTS PASSED", everything works.

**Progressive assertion with labels**: Each test section prints a label (`'Write OK'`, `'Read OK'`, etc.) after its assertions pass. If the script crashes, the last printed label tells you which section succeeded, so you can infer where the failure is.

**Tombstone convention**: Deletion is represented as `value=None` rather than a separate delete operation. This is the standard LSM-tree approach — deletes are writes of a special marker that gets cleaned up during compaction.

**Timestamp-based conflict resolution**: When merging SSTables with duplicate keys, the entry with the highest timestamp wins (line ~87: `assert shared[0].value == 'val_4'` because `float(4)` is the highest timestamp). This is a last-writer-wins strategy.

## Dependencies

**Imports:**
- `tempfile`, `os`, `sys` — standard library for temp directories and paths
- `from sstable import *` — the entire public API of the implementation module

**Imported by:** Nothing. This is a leaf test script.

## Flow

The script executes top-to-bottom in a single temp directory:

1. **Write** → Create `001.sst` with 5 fruit entries, verify metadata
2. **Read** → Point lookup for `'cherry'` (hit) and `'fig'` (miss)
3. **Range scan** → `range_scan('banana', 'elderberry')` returns 3 entries (banana, cherry, date — endpoint-exclusive on the upper bound)
4. **Write2** → Create `002.sst` with an update (`apple` → `'green'`), a tombstone (`banana` → `None`), and a new key (`fig`)
5. **Merge** → Merge `002.sst` (higher priority, listed first) with `001.sst`, tombstones removed. Result: 5 entries (apple=green, cherry, date, elderberry, fig). Banana is gone.
6. **Compaction** → Size-tiered compaction with `min_threshold=2`, triggers when 2+ SSTables exist
7. **Tombstone** → Verify a tombstone entry is readable (value is None, but entry exists)
8. **Empty SSTable** → Write/read a table with zero entries
9. **Single entry** → Write/read a table with exactly one entry, verify min_key == max_key
10. **Multi-way merge** → 5 SSTables each with a unique key and a shared key `'shared'`. Verifies dedup: `'shared'` appears once with the highest-timestamp value
11. **Leveled compaction** → 3 L0 SSTables with `l0_compaction_trigger=2`. Verifies compaction fires and produces L1 output (`result[0].level == 1`)

## Invariants

- **Sorted key order**: `SSTableWriter.add` must be called with keys in sorted order (the test does this — apple < banana < cherry < date < elderberry).
- **Tombstone semantics**: `value=None` is a tombstone. `get()` still returns the entry (it doesn't hide it), but `merge_sstables` with `remove_tombstones=True` drops it.
- **Merge priority by list position**: `merge_sstables([reader2, reader], ...)` — `reader2` (the newer table) is listed first. Combined with timestamp-based resolution, this means the first reader's entries win when timestamps are higher.
- **Range scan is start-inclusive, end-exclusive**: `range_scan('banana', 'elderberry')` returns banana, cherry, date (3 entries, not 4).
- **Leveled compaction promotes to L1**: After compaction, output SSTables have `level == 1`.

## Error Handling

There is none, deliberately. Every assertion failure crashes the script with a traceback. Some assertions include diagnostic messages (e.g., `f'range_scan len={len(results)}: {[e.key for e in results]}'`) to help debug failures, but there's no try/except, no graceful degradation, no retry logic. This is appropriate for a verification script.

---

## Topics to Explore

- [file] `sstable-and-compaction/sstable.py` — The implementation being tested: how SSTable file format, block structure, sparse index, and merge logic actually work
- [function] `sstable-and-compaction/sstable.py:merge_sstables` — The N-way merge algorithm: how it does a k-way sorted merge with timestamp-based dedup and tombstone removal
- [file] `sstable-and-compaction/tester_test_sstable.py` — Likely a more comprehensive pytest-style test suite for the same module, worth comparing coverage
- [file] `log-structured-merge-tree/lsm.py` — The full LSM tree implementation that uses SSTables as its on-disk layer, showing how memtable flushes produce SSTables and compaction is triggered
- [general] `leveled-vs-size-tiered-compaction` — The two compaction strategies tested here have very different tradeoffs in write amplification, space amplification, and read performance (DDIA Chapter 3)

## Beliefs

- `sstable-merge-uses-timestamp-resolution` — `merge_sstables` resolves duplicate keys by keeping the entry with the highest timestamp (last-writer-wins)
- `sstable-tombstone-is-null-value` — Tombstones are represented as entries with `value=None`, not as a separate deletion record type
- `sstable-range-scan-end-exclusive` — `range_scan(start, end)` is inclusive on `start` and exclusive on `end`
- `compaction-manager-supports-two-strategies` — `CompactionManager` implements both `'size_tiered'` and `'leveled'` compaction strategies
- `leveled-compaction-promotes-to-l1` — After leveled compaction runs on L0 SSTables, the output has `level == 1`

