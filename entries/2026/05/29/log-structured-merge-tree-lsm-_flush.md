# Function: _flush in log-structured-merge-tree/lsm.py

**Date:** 2026-05-29
**Time:** 07:02

## `LSMTree._flush` — Flush Memtable to SSTable

### Purpose

`_flush` persists the current in-memory sorted buffer (the memtable) to disk as a new SSTable file. This is the core mechanism that converts volatile writes into durable storage. It exists because the memtable is bounded in size — once it hits `memtable_threshold` entries, the data must move to disk to keep memory usage bounded while preserving all writes.

In the LSM tree architecture from DDIA Chapter 3, this is the transition from the "in-memory balanced tree" layer to the "on-disk sorted segment" layer.

### Contract

**Preconditions:**
- `self._dir` exists on the filesystem (guaranteed by `__init__` calling `os.makedirs`)
- `self._wal` is open and writable
- `self._seq` is monotonically increasing (no two SSTables share a sequence number)

**Postconditions:**
- `self._memtable` is a fresh, empty `SortedDict`
- A new `.sst` file exists on disk containing all entries from the old memtable
- The new `SSTable` is appended to `self._sstables` (newest-last ordering preserved)
- The WAL is truncated (safe because data is now durable in the SSTable)
- If the SSTable count reaches `_compaction_threshold`, compaction runs immediately

**Invariant:** After `_flush`, the union of memtable + immutable memtables + SSTables still contains every non-compacted write. No data is lost.

### Parameters

None — this is a private method operating entirely on instance state.

### Return Value

`None`. The caller (`put`, `delete`, `close`) does not check a return value. Success is implicit; failure would raise an exception from I/O operations.

### Algorithm

1. **Guard clause:** If the memtable is empty, return immediately. This prevents creating zero-entry SSTable files (which would waste disk and confuse compaction).

2. **Freeze the memtable:** Save a reference to the current `SortedDict` in `frozen`, then replace `self._memtable` with a new empty `SortedDict`. This is a pointer swap — `frozen` and the new memtable are independent objects. New writes arriving after this point go into the fresh memtable.

3. **Allocate a sequence number:** `_next_seq()` returns the current `self._seq` and increments it. This gives the SSTable a unique, monotonically increasing identity used for both the filename and merge ordering during reads/compaction.

4. **Construct the file path:** Format as `sst_000042.sst` (zero-padded to 6 digits). The zero-padding ensures lexicographic sort of filenames matches numeric sort — important for `_load_existing_sstables` which uses `sorted()` on filenames at recovery time.

5. **Write the SSTable:** `SSTable.write()` serializes all entries in sorted order (guaranteed by `SortedDict`) into a binary file with a sparse index footer, and returns the in-memory `SSTable` handle with the sparse index already populated.

6. **Register the SSTable:** Append to `self._sstables`. Since SSTables are appended in creation order and sequence numbers are monotonic, the newest-last invariant is maintained.

7. **Truncate the WAL:** The WAL's purpose is crash recovery for unflushed memtable data. Once the data is persisted to an SSTable, the WAL entries are redundant and can be discarded. `WAL.truncate()` opens the file in write mode (zeroing it) then reopens in append mode.

8. **Trigger compaction if needed:** If the total SSTable count meets or exceeds `_compaction_threshold`, call `self.compact()` to merge all SSTables into one, eliminating duplicates and tombstones. This is size-triggered compaction (as opposed to tiered or leveled strategies).

### Side Effects

| Side Effect | Details |
|---|---|
| **Disk I/O** | Creates a new `.sst` file via `SSTable.write` |
| **WAL truncation** | Zeros the WAL file — data only in the WAL before flush is now only in the SSTable |
| **State mutation** | Replaces `_memtable`, increments `_seq`, appends to `_sstables` |
| **Cascading compaction** | May trigger `compact()`, which deletes old SSTable files and creates a merged one |

### Error Handling

There is no explicit error handling. If `SSTable.write` or the filesystem operations fail (disk full, permission denied), the exception propagates to the caller (`put`, `delete`, or `close`). This leaves the system in a partially-flushed state:

- The memtable has already been swapped to empty, so the data in `frozen` would be lost if the write fails partway through.
- The WAL has not yet been truncated (truncation happens after the write), so WAL replay at recovery would restore the data — **but only if the memtable swap and failed SSTable write don't leave a corrupt partial file on disk**.

This is a known simplification: a production LSM tree would use immutable memtables (note `_immutable_memtables` exists but `_flush` doesn't use it) and atomic file rename to handle this window.

### Usage Patterns

`_flush` is called from three sites:

- **`put()`** — after inserting a key, if `len(self._memtable) >= self._threshold`
- **`delete()`** — same threshold check after writing a tombstone
- **`close()`** — unconditionally, to persist any remaining memtable entries before shutdown

It is a private method (underscore prefix). External callers are expected to use `put`/`delete`/`close` and let flush happen automatically.

### Dependencies

| Dependency | Role |
|---|---|
| `SortedDict` (sortedcontainers) | Provides the sorted memtable; `list(frozen.items())` yields entries in key order |
| `SSTable.write` | Serializes entries to the binary SSTable format with sparse index |
| `WAL.truncate` | Clears the write-ahead log after successful persistence |
| `os.path.join` | Constructs the SSTable file path |
| `self.compact` | Called conditionally to merge SSTables |

### Notable Assumptions

1. **Single-threaded access.** The memtable swap (`frozen = self._memtable; self._memtable = SortedDict()`) is not atomic. Concurrent `put()` calls during flush would race on the memtable reference.
2. **`_immutable_memtables` is unused during flush.** The frozen memtable is written directly to disk rather than being staged as an immutable memtable first. This means reads during flush could miss keys that are between the old memtable (now `frozen`, not checked by `get`) and the new SSTable (not yet written). A production implementation would append `frozen` to `_immutable_memtables` before writing.
3. **WAL covers exactly one memtable.** Truncating the WAL assumes all WAL entries correspond to the flushed memtable. If a crash occurs after the memtable swap but before WAL truncation, replay would re-insert entries into a fresh memtable — correct behavior.
4. **`SSTable.write` is synchronous and infallible.** No retry or partial-write recovery.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:compact` — The compaction strategy this flush triggers; understanding the merge semantics is essential to understanding how flush fits into the write lifecycle
- [function] `log-structured-merge-tree/lsm.py:SSTable.write` — The binary serialization format and sparse index construction that `_flush` delegates to
- [function] `log-structured-merge-tree/lsm.py:WAL.truncate` — The truncation mechanism and its crash-safety implications (open-as-write then reopen-as-append)
- [general] `immutable-memtable-gap` — The `_immutable_memtables` list exists and is checked during `get()` and `range_scan()`, but `_flush` never populates it — this is a consistency gap worth investigating
- [file] `log-structured-merge-tree/test_lsm.py` — Test coverage for flush behavior, especially edge cases like flushing an empty memtable and crash recovery scenarios

---

## Beliefs

- `flush-creates-sequential-sstables` — Each `_flush` call creates an SSTable file with a strictly increasing sequence number, preserving newest-last ordering in `self._sstables`
- `wal-truncated-after-sstable-write` — The WAL is truncated only after the SSTable is fully written and appended to the in-memory list, ensuring crash recovery can replay unflushed data
- `flush-skips-immutable-memtable-stage` — `_flush` writes directly to disk without staging the frozen memtable in `_immutable_memtables`, creating a brief window where in-flight keys are invisible to `get()`
- `compaction-triggered-by-sstable-count` — Compaction runs automatically when `len(self._sstables) >= self._compaction_threshold`, which defaults to 4

