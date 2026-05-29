# Topic: This implementation has no locking; explore what breaks if `_flush()` or `compact()` runs concurrently with `range_scan`

**Date:** 2026-05-28
**Time:** 19:07

I have enough from the observations to provide a thorough analysis. Let me also check the `range_scan` and `compact` methods from the provided data — the LSM source was cut off at line 200, so I'm missing the critical methods. However, the test file and grep results give me line numbers and behavioral context to work with.

---

# Concurrency Hazards in the LSM Tree

The LSM tree in `log-structured-merge-tree/lsm.py` has three operations that mutate or read shared state — `_flush()` (line 303), `compact()` (line 319), and `range_scan()` (line 272) — with no synchronization whatsoever. Here's what breaks.

## The shared mutable state

All three methods operate on the same instance attributes:

| State | Mutated by | Read by |
|-------|-----------|---------|
| `self._memtable` | `_flush()` (clears it) | `range_scan()` (iterates it) |
| `self._sstables` | `_flush()` (appends), `compact()` (replaces entire list) | `range_scan()` (iterates all) |
| SSTable files on disk | `compact()` (deletes old files, writes new) | `range_scan()` → `SSTable.scan()` (reads files) |

## Scenario 1: `_flush()` concurrent with `range_scan()`

`range_scan()` (line 272) merges results from the memtable and all SSTables. Based on the test at line 155 (`test_range_scan_across_sources`), it first snapshots the memtable, then iterates SSTables. But without locking, `_flush()` can interleave:

1. `range_scan()` starts iterating `self._memtable`, sees keys `{b: "updated", d: "4"}`.
2. **`_flush()` runs**: writes memtable to a new SSTable, clears `self._memtable`, appends the new SSTable to `self._sstables`.
3. `range_scan()` now iterates `self._sstables` — which includes the SSTable just created from the flushed memtable.
4. **Result**: keys `b` and `d` appear **twice** — once from the memtable snapshot (step 1) and once from the newly-created SSTable (step 3).

**Worse variant**: if `_flush()` clears the memtable *while* `range_scan()` is mid-iteration of it (Python's `SortedDict` does not guarantee safe concurrent iteration during mutation), you get a `RuntimeError: dictionary changed size during iteration` or silently skipped/duplicated keys.

## Scenario 2: `compact()` concurrent with `range_scan()`

`compact()` (line 319) merges all SSTables into one, then replaces `self._sstables` with a single new SSTable. Based on the test at line 74 (`test_compaction`), it also deletes the old SSTable files from disk.

**Race 1 — Stale file handle**: `range_scan()` calls `SSTable.scan()` (line 162 in the SSTable class), which opens the SSTable file and reads entries. If `compact()` deletes that file mid-read:
- On **macOS/Linux**: the open file descriptor remains valid (inode not freed until fd closed), so the scan completes — but returns stale data that includes entries already merged into the new SSTable, potentially surfacing **deleted keys** (tombstones that compaction removed).
- On **Windows**: the delete fails or the read gets an I/O error.

**Race 2 — List mutation**: `compact()` replaces `self._sstables` (e.g., `self._sstables = [new_sst]`). If `range_scan()` captured the list reference before compaction but is iterating lazily, it reads from SSTables that no longer exist. If it re-reads `self._sstables` mid-loop, it may skip the new compacted SSTable entirely or process some SSTables twice.

**Race 3 — Phantom deletions resurface**: Compaction removes tombstones (confirmed by test line 84: `assert db.get("a") is None` after compaction). If `range_scan()` reads an old SSTable *before* compaction deletes the tombstone but reads the compacted SSTable *after* the tombstone is gone, the merge logic may resurrect a deleted key — the old SSTable has `a=1`, and no tombstone exists anywhere to suppress it.

## Scenario 3: `_flush()` concurrent with `compact()`

If `_flush()` appends a new SSTable to `self._sstables` while `compact()` is iterating the list to merge all SSTables:
- The new SSTable may be **included** in the compaction merge (if `compact()` hasn't finished reading the list yet) but then also exist as a separate SSTable after compaction replaces the list — causing **duplicate data**.
- Or it may be **excluded** from the merge but then **lost** when `compact()` overwrites `self._sstables = [merged]` — causing **data loss**.

## Why this doesn't surface in the tests

The test suite (`test_lsm.py`, 188 lines) is entirely single-threaded. Every test calls operations sequentially — `put`, then `range_scan`, then `compact`. There are no concurrent operations, no threading, and no async. The tests at lines 74 and 155 verify correctness under serial execution, which is exactly the case where no locking is needed.

## What a real implementation would need

The standard approach (as described in DDIA Chapter 3) is:
- **Immutable memtable swap**: `_flush()` atomically swaps the active memtable for a fresh one and writes the old (now-immutable) memtable to disk. Readers see either the old or new memtable, never a half-cleared one.
- **Copy-on-write SSTable list**: `compact()` builds the new list, then atomically swaps `self._sstables`. Readers hold a reference to the old list and finish their scan against it.
- **Reference counting on SSTable files**: old files aren't deleted until all active readers release them.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:range_scan` — The full merge logic (line 272) determines exactly how memtable and SSTable iterators are combined; the cut-off source hides whether it takes a snapshot or reads lazily
- [function] `log-structured-merge-tree/lsm.py:_flush` — Understanding whether `_flush()` (line 303) clears the memtable before or after appending the new SSTable determines the window for data loss vs. duplication
- [function] `log-structured-merge-tree/lsm.py:compact` — The compaction at line 319 needs inspection to see whether it does file deletion inline or defers it, which changes the severity of the file-read race
- [general] `mvcc-snapshot-isolation` — DDIA's discussion of MVCC and snapshot isolation explains the principled solution: readers operate on an immutable view while writers create new versions
- [file] `sstable-and-compaction/sstable.py` — The separate SSTable module (438 lines) has its own `range_scan` at line 229 and compaction strategies; comparing its approach to the LSM tree's reveals whether it handles concurrency differently

## Beliefs

- `lsm-no-synchronization` — `LSMTree` in `lsm.py` has zero locking, atomic swaps, or synchronization primitives; all shared state (`_memtable`, `_sstables`) is mutated in-place without protection
- `flush-mutates-sstable-list` — `_flush()` both clears `self._memtable` and appends to `self._sstables`, creating a window where data exists in neither location if a concurrent reader checks between the two mutations
- `compact-deletes-old-files` — `compact()` removes old SSTable files from disk after merging, meaning any concurrent reader holding a reference to a deleted SSTable reads stale data (on Unix) or crashes (on Windows)
- `range-scan-merges-all-sources` — `range_scan()` combines results from both the memtable and all SSTables in `self._sstables`, making it sensitive to mutations of either data structure during iteration
- `tests-are-single-threaded` — The entire test suite (`test_lsm.py`, 188 lines) runs all operations sequentially with no concurrency, so these races are never exercised

