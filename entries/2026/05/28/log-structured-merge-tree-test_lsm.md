# File: log-structured-merge-tree/test_lsm.py

**Date:** 2026-05-28
**Time:** 18:25

# `log-structured-merge-tree/test_lsm.py`

## Purpose

This is the test suite for an LSM Tree (Log-Structured Merge Tree) storage engine implementation. It validates the `LSMTree` class from `lsm.py` against the behavioral contract described in DDIA Chapter 3 — the core read/write path, flush-to-disk lifecycle, multi-SSTable reads, compaction, crash recovery via WAL replay, and range scans that merge across storage layers.

It serves as both a correctness harness and a living specification: each test encodes a specific invariant of LSM tree behavior.

## Key Components

### Test Functions

| Test | What it exercises |
|------|-------------------|
| `test_basic_crud` | Full lifecycle: put, get, update (overwrite), delete (tombstone), range scan — all in one flow that crosses a flush boundary |
| `test_flush_and_sstable_reads` | Memtable auto-flush at threshold; reads transparently fall through to SSTables |
| `test_multiple_sstables_newest_wins` | Write ordering across multiple SSTables — newer SSTable shadows older for the same key |
| `test_compaction` | Explicit `compact()` merges N SSTables into 1, garbage-collects tombstoned keys |
| `test_crash_recovery` | WAL replay: writes without `close()`, reopen, verify data survived |
| `test_large_dataset` | 10K key scale test with `memtable_threshold=500` and `compaction_threshold=10`, exercising automatic compaction |
| `test_edge_cases` | Boundary conditions: empty db, single key, delete-nonexistent, 100× overwrite, empty range scan |
| `test_range_scan_across_sources` | Range scan merges memtable and SSTable data, with memtable values correctly overriding SSTable values for the same key |

### `LSMTree` Constructor Parameters (as revealed by tests)

- **`directory`**: path to the data directory (all tests use `tempfile.TemporaryDirectory`)
- **`memtable_threshold`**: number of entries that triggers an auto-flush to SSTable
- **`compaction_threshold`**: number of SSTables that triggers (or allows) compaction — set to `100` in several tests to *prevent* auto-compaction so the test can control it manually

## Patterns

**Temp directory isolation** — every test creates its own `tempfile.TemporaryDirectory()` via a context manager. No shared state between tests; each gets a clean filesystem. This is critical because LSM trees are inherently stateful on disk.

**Threshold manipulation for test control** — tests set `memtable_threshold` low (2–3) to force flushes at predictable points, and `compaction_threshold` high (100) to suppress automatic compaction. This lets each test exercise one specific behavior in isolation.

**Crash simulation via reopen** — `test_crash_recovery` skips `close()` and constructs a new `LSMTree` on the same directory. The WAL is the durability mechanism; the test proves unflushed memtable data survives via replay.

**Dual runner** — the `if __name__ == "__main__"` block provides a standalone runner that catches and reports per-test exceptions, printing PASS/FAIL. This makes it runnable without pytest while remaining pytest-compatible (all functions start with `test_`).

## Dependencies

**Imports:**
- `tempfile` — stdlib, for isolated test directories
- `lsm.LSMTree` — the system under test

**Imported by:** Nothing — this is a leaf test file.

## Flow

Each test follows the same pattern:

1. Create a temp directory
2. Instantiate `LSMTree` with tuned thresholds
3. Issue a sequence of `put`/`get`/`delete`/`range_scan`/`compact` calls
4. Assert expected outcomes at each step
5. Call `db.close()` (except in the crash recovery test, deliberately)

The ordering within `test_basic_crud` is particularly instructive — it crosses a flush boundary mid-test (the 3rd `put` hits the threshold), then continues operating, proving the read path transparently merges memtable and SSTable data.

## Invariants

- **Newest-write-wins**: if the same key is written multiple times (across memtable, across SSTables, or both), `get` returns the most recent value. Tested explicitly in `test_multiple_sstables_newest_wins` and `test_range_scan_across_sources`.
- **Tombstone semantics**: `delete(k)` causes `get(k)` to return `None`, even if `k` exists in an older SSTable. After compaction, the tombstone and the original entry are both purged.
- **Range scan is half-open**: `range_scan("apple", "date")` returns `apple` and `cherry` but not `date` — this is a `[start, end)` interval.
- **Compaction reduces SSTable count to 1**: `test_compaction` asserts exactly 3 SSTables before and 1 after.
- **WAL guarantees durability of unflushed writes**: data written to the memtable (never flushed) survives a simulated crash.
- **Auto-flush triggers at exactly `memtable_threshold` entries**: tests rely on precise control of when flushes happen based on the count of `put` calls.

## Error Handling

Minimal — the tests assert expected values and let pytest's `AssertionError` propagate naturally. The standalone runner at the bottom wraps each test in a try/except to print tracebacks without aborting the suite, but this is a convenience, not a production error-handling pattern. No tests verify error conditions (e.g., writing to a read-only directory, corrupted WAL).

## Topics to Explore

- [file] `log-structured-merge-tree/lsm.py` — The implementation being tested: memtable, SSTable format, WAL, compaction logic, and the read merge path
- [function] `log-structured-merge-tree/lsm.py:range_scan` — How the merge across memtable + multiple SSTables works, especially the deduplication/ordering logic
- [function] `log-structured-merge-tree/lsm.py:compact` — The compaction algorithm: how it merges SSTables, resolves duplicates, and strips tombstones
- [file] `sstable-and-compaction/sstable.py` — A related implementation in the repo focused specifically on SSTable format and compaction, likely a different factoring of similar concepts
- [general] `lsm-crash-recovery-wal-format` — What the WAL on-disk format looks like, and how replay handles partial writes or corruption

## Beliefs

- `lsm-range-scan-half-open` — `range_scan(start, end)` returns keys in `[start, end)` — inclusive of start, exclusive of end
- `lsm-memtable-threshold-triggers-flush` — When the number of entries in the memtable reaches `memtable_threshold`, the memtable is automatically flushed to a new SSTable on disk
- `lsm-compaction-merges-all-to-one` — `compact()` merges all existing SSTables into a single SSTable, discarding tombstoned entries
- `lsm-wal-replays-on-reopen` — Constructing a new `LSMTree` on an existing directory replays the WAL to recover unflushed memtable state
- `lsm-newest-write-wins-across-sstables` — When the same key exists in multiple SSTables, the value from the most recently created SSTable is returned

