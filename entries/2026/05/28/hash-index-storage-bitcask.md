# File: hash-index-storage/bitcask.py

**Date:** 2026-05-28
**Time:** 18:08

# `hash-index-storage/bitcask.py`

## Purpose

This file implements a **Bitcask-style append-only key-value storage engine** — the canonical hash-indexed log structure described in DDIA Chapter 3. It owns the full storage lifecycle: writing key-value records to append-only data files, maintaining an in-memory hash index (`keydir`) for O(1) lookups, and compacting old files to reclaim space from stale/deleted entries.

Bitcask is the simplest practical storage engine that separates the write path (sequential append) from the read path (random seek via index), trading RAM for read performance.

## Key Components

### Constants

- **`HEADER_FORMAT = "<dII"`** — Little-endian binary record header: 8-byte `double` timestamp, two 4-byte `uint32` for key and value lengths. Total: 16 bytes (`HEADER_SIZE`).
- **`HINT_FORMAT = "<IQId"`** — Hint file entry: `file_id` (u32), `offset` (u64), `size` (u32), `timestamp` (double). 24 bytes (`HINT_HEADER_SIZE`). Used to speed up index rebuilds by avoiding full data file scans.

### `KeyEntry` (dataclass)

The in-memory index record. Maps a key to its physical location: which `file_id`, at what byte `offset`, how many bytes (`size`), and the write `timestamp`. This is the value type in the `keydir` hash map.

### `BitcaskStore`

The main engine. Constructor parameters:

| Param | Default | Role |
|-------|---------|------|
| `data_dir` | required | Directory for all `.data` and `.hint` files |
| `max_file_size` | 10 MB | Threshold for rotating to a new data file |
| `sync_writes` | `True` | Whether to `fsync` after every write (durability vs. throughput) |

**Core state:**
- `keydir: dict[str, KeyEntry]` — the in-memory hash index. This is the only index; every live key has exactly one entry.
- `file_handles: dict[int, file]` — pool of read-mode file handles, keyed by file ID.
- `active_file` / `active_file_id` — the single writable file (opened in append mode).

## Patterns

1. **Append-only log with in-memory index** — Writes never overwrite; they append a new record and update the `keydir` pointer. This makes writes sequential and crash-safe (partial writes are just trailing garbage).

2. **Tombstone deletion** — `delete()` writes a record with an empty value (`""`) and removes the key from `keydir`. During compaction and index rebuild, zero-length values are treated as tombstones.

3. **File rotation** — When the active file exceeds `max_file_size`, it becomes immutable and a new file takes over. Only immutable files are candidates for compaction.

4. **Hint files** — After compaction, a `.hint` file is written alongside each merged `.data` file. Hint files contain only index metadata (no values), so startup recovery can skip the expensive full-scan path via `_load_hint_file` instead of `_scan_data_file`.

5. **Single-writer model** — There is exactly one active (writable) file at any time. All older files are read-only. This avoids write contention without locks.

## Dependencies

**Imports:** Only stdlib — `os`, `struct`, `time`, `dataclasses`, `typing`. No external dependencies. This is intentional for a reference implementation.

**Imported by:** Test files in both `hash-index-storage/` and `log-structured-hash-table/`, suggesting the latter reuses or mirrors this implementation for comparison.

## Flow

### Write path (`put`)

```
put(key, value)
  → _maybe_rotate()          # check if active file is full
  → _write_record(key, value) # pack header + key + value, append, flush, fsync
  → update keydir[key]        # point to new record location
```

### Read path (`get`)

```
get(key)
  → keydir.get(key)           # O(1) hash lookup
  → _read_record(fid, off, sz) # seek + read from the right data file
  → assert key matches        # integrity check
  → return value (or None if tombstone)
```

### Startup recovery (`__init__`)

```
__init__
  → _find_file_ids()          # scan directory for *.data files
  → _rebuild_index(ids)       # for each file: use hint file if available, else scan data file
  → _open_active_file()       # open the highest-numbered file for appending
```

### Compaction (`compact`)

1. Identify immutable files (everything except active).
2. Scan them chronologically, keeping only the latest record per key (tombstones evict).
3. Filter further: skip keys whose `keydir` entry points to the active file (those immutable records are already stale).
4. Write surviving records into new merged file(s) with new file IDs.
5. Write hint files for the merged files.
6. Delete old immutable data and hint files.
7. Renumber the active file to sit after the merged files and reopen it.

## Invariants

- **Every live key has exactly one `keydir` entry**, pointing to the most recent non-tombstone record. Stale records exist on disk but are invisible to reads.
- **Only the active file is opened for writing.** All other files are read-only.
- **Tombstones are zero-length values.** Both `_scan_data_file` and `compact` treat `val_size == 0` as a delete marker; `get` treats `value == ""` as `None`.
- **File IDs are monotonically increasing.** `_find_file_ids` sorts them; the active file is always the highest. After compaction, merged files get IDs above the old active, and the active file is renumbered above those.
- **Hint files are authoritative for index rebuild.** If a `.hint` file exists for a file ID, `_rebuild_index` trusts it and skips the data scan. This means hint files must be written atomically with their data files during compaction — there's no checksum validation here.
- **Records are self-describing.** The header contains key and value sizes, so records can be scanned sequentially without an external schema.

## Error Handling

Minimal — appropriate for a reference implementation, but notable gaps:

- **No CRC / checksum.** Corrupt records will be read and returned without detection. The only integrity check is the `assert read_key == key` in `get()`, which catches index/data mismatches but not bit-rot.
- **No partial-write recovery.** If a write is interrupted mid-record (crash after partial `write()` but before `fsync`), `_scan_data_file` will hit a short read on the header (`len(header_data) < HEADER_SIZE`) and silently stop scanning — truncating any records that followed in that file. This is acceptable for Bitcask's crash model (the incomplete record is simply lost).
- **File handle leaks on error.** If `_write_record` or `compact` raises mid-operation, open file handles in `file_handles` and temporary files in `compact` are not cleaned up. There's no `__enter__`/`__exit__` context manager protocol.
- **`fsync` is per-record when `sync_writes=True`.** This is the durable-by-default choice, at significant write throughput cost.

## Topics to Explore

- [file] `log-structured-hash-table/bitcask.py` — Compare with this implementation to understand what differs between the two hash-indexed variants in the repo
- [function] `hash-index-storage/bitcask.py:compact` — The most complex method; trace how file IDs are reassigned and how the active file survives renumbering
- [file] `sstable-and-compaction/sstable.py` — The sorted-string-table engine adds ordering and merge-sort compaction on top of the log-structured foundation
- [general] `bitcask-crash-recovery` — What happens when a crash occurs mid-compaction? The delete-then-rename sequence has a window where data could be lost
- [file] `log-structured-merge-tree/lsm.py` — LSM trees build on the same append-only log idea but add a memtable and tiered compaction strategy

## Beliefs

- `bitcask-tombstone-is-empty-string` — Deletion is represented by writing a record with an empty-string value; both `_scan_data_file` (checks `val_size == 0`) and `get` (checks `value == ""`) use this convention
- `bitcask-single-active-writer` — Exactly one data file is open for writes at any time; all others are immutable and read-only
- `bitcask-hint-files-skip-scan` — When a `.hint` file exists for a file ID, `_rebuild_index` loads it instead of scanning the data file, trading disk space for faster startup
- `bitcask-no-checksum-validation` — Records have no CRC or integrity checksum; the only corruption guard is the `assert read_key == key` in `get()`
- `bitcask-compaction-renumbers-active` — After compaction, the active file is renamed to a higher file ID to maintain the monotonically increasing ID invariant

