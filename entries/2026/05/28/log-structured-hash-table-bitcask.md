# File: log-structured-hash-table/bitcask.py

**Date:** 2026-05-28
**Time:** 18:21

## Purpose

This file implements a **Bitcask-style log-structured hash table** — the storage engine described in DDIA Chapter 3 as the simplest viable approach to indexing. It owns the full lifecycle of an append-only key-value store: writing records to segmented log files, maintaining an in-memory hash index for O(1) lookups, compacting frozen segments to reclaim space, and recovering state on restart.

Bitcask (originally from Riak) makes one core trade-off: all keys must fit in memory (the hash index), but in exchange you get single-disk-seek reads and sequential-write-only appends.

## Key Components

### Constants and Wire Format

```
TOMBSTONE = b"__BITCASK_TOMBSTONE__"
HEADER_FMT = "!III"   # network-byte-order: crc32 (4B), key_size (4B), value_size (4B)
HEADER_SIZE = 12
```

Each record on disk is: `[crc32 | key_len | val_len | key_bytes | val_bytes]`. The CRC covers only the payload (key + value bytes), not the header — important to know when debugging integrity checks.

Hint files use a smaller format (`!II` — key_size + offset) that lets recovery skip reading full values.

### `BitcaskStore`

The main class. Its state consists of:

- **`_index`**: `dict[str, tuple[str, int]]` — maps every live key to `(segment_file_path, byte_offset)`. This is the "hash table" in "log-structured hash table."
- **`_active_file`** / **`_active_path`**: the currently-writable segment, opened in append mode.
- **`_segment_counter`**: monotonically increasing ID that determines segment filenames (`segment_000003.dat`).
- **`_file_handles`**: a cache of read-mode file handles (though `get()` actually opens fresh handles — more on that below).

### Core operations

| Method | Contract |
|--------|----------|
| `put(key, value)` | Appends record to active segment, updates index, auto-rotates and auto-compacts. Returns byte offset. |
| `get(key)` | Single index lookup + single disk seek. Returns `None` for missing keys, raises `CorruptionError` on integrity failure. |
| `delete(key)` | Writes a tombstone record, removes key from index. Returns whether the key existed. |
| `compact()` | Merges all frozen segments into one new segment, deletes old files, updates index. Returns count of stale records removed. |

### `SegmentInfo`

A simple data class returned by `segments()` for introspection — exposes segment ID, path, size, record count, and whether it's the active segment.

## Patterns

**Append-only log with in-memory index** — the textbook Bitcask pattern. Writes never modify existing data; all mutations are appends. The index is ephemeral and rebuilt from the log on startup.

**Segment rotation** — when the active segment exceeds `max_segment_size` (default 1MB), it's frozen and a new segment begins. This bounds individual file size and enables per-segment compaction.

**Tombstone deletion** — deletes don't remove data; they append a sentinel value (`TOMBSTONE`). The key is immediately removed from the index, but the tombstone persists on disk until compaction merges it away.

**Hint files** — an optimization for recovery. Instead of scanning full segment data, a hint file stores just `(key_size, offset, key)` tuples, letting `_recover()` skip reading values entirely. Only created for frozen segments via `create_hint_files()`.

**Auto-compaction** — `put()` checks if the number of frozen segments reaches `auto_compact_threshold` (default 5) and triggers compaction automatically.

## Dependencies

**Imports**: Only stdlib — `os` (filesystem), `struct` (binary packing), `zlib` (CRC32), `typing` (annotations). No external dependencies.

**Imported by**: Test files in both `log-structured-hash-table/` and `hash-index-storage/` — this module appears to be the reference implementation shared by two exercise directories.

## Flow

### Write path (`put`)

1. Check if active segment is full → `_rotate_segment()` if so
2. `_write_record()`: encode key to UTF-8, compute CRC32 over `key_bytes + value`, pack header, append header + payload, flush
3. Invalidate any cached read handle for the active path (stale file position)
4. Update `_index[key] = (active_path, offset)`
5. If frozen segment count ≥ threshold → `compact()`

### Read path (`get`)

1. Index lookup — `O(1)`, returns `(seg_path, offset)` or `None`
2. Open a **fresh** file handle (not the cached one from `_file_handles`), seek to offset
3. Read header, then payload
4. Verify CRC32 — raise `CorruptionError` on mismatch
5. Return `payload[key_size:]` (the value portion)

Note: `get()` opens a new file handle every call rather than using `_get_read_handle()`. The cached handle pool (`_file_handles`) exists but is only used... nowhere in practice for reads. This is arguably a consistency issue — the cache is maintained during `put()` (entries are popped) and `compact()` (entries are closed), but `get()` sidesteps it entirely.

### Recovery path (`_recover`)

1. List all `segment_*.dat` files, sort by ID
2. For each segment (oldest first): load from hint file if available, otherwise full scan
3. During scan: tombstones remove keys from index, live records add/overwrite
4. The highest-ID segment becomes the active segment, opened in append mode

Ordering matters: scanning oldest-to-newest ensures the latest write for any key wins.

### Compaction (`compact`)

1. Scan all frozen segments, collecting `(key, seg_path, offset)` for live entries
2. Filter: only keep entries whose index still points to that exact `(seg_path, offset)` — this discards entries superseded by writes to the active segment
3. Write survivors to a new compacted segment with fresh CRC values
4. Update index atomically (dict.update)
5. Close cached handles, delete old segment files and hint files
6. Rename the old active segment to a new higher-ID path, update all index references, reopen for appending

The rename-and-reindex step for the active segment is the trickiest part — it ensures the active segment always has the highest ID, preserving the recovery invariant.

## Invariants

1. **The active segment always has the highest segment ID.** Recovery depends on this — it opens the last segment for appending.
2. **The index is the sole authority for live keys.** A key exists if and only if it's in `_index`. Disk may contain stale or tombstoned records.
3. **Segment files are append-only while active.** Only compaction writes a new file from scratch; the active file is only ever appended to.
4. **Scanning order is oldest-to-newest.** Both `_recover()` and `compact()` process segments in ascending ID order so later writes overwrite earlier ones in the index.
5. **CRC covers payload only (key + value), not the header.** The header contains the CRC itself, so it can't be part of the checksum input.
6. **Tombstones are never written to hint files.** `create_hint_files()` explicitly skips them, and only writes entries whose index still points to that segment+offset.

## Error Handling

- **CRC mismatch on read (`get`)**: raises `CorruptionError` with offset and expected-vs-actual values. This is a hard failure — no fallback.
- **CRC mismatch on recovery scan**: silently stops scanning that segment (`break`). Partial writes at the tail of a segment (from a crash) are treated as the end of valid data. This is the correct behavior — a crash during append leaves a partial record at the end, which should be ignored.
- **Short reads (incomplete header/payload)**: treated as end-of-file during scans, raised as `CorruptionError` during `get()`. The asymmetry is intentional — scans tolerate crash artifacts, but explicit reads of indexed offsets should not encounter them.
- **File handle leaks**: `close()` cleans up, but if the process exits without calling `close()`, OS-level cleanup handles it. The `_file_handles` cache could accumulate handles for deleted segments if reads happened before compaction, though in practice `get()` doesn't use the cache.

## Topics to Explore

- [file] `log-structured-hash-table/test_bitcask.py` — Test cases reveal the expected behavioral contracts, especially around compaction correctness and crash recovery
- [file] `hash-index-storage/bitcask.py` — Compare with the other bitcask implementation in the repo to understand what design decisions differ between the two exercise variants
- [file] `sstable-and-compaction/sstable.py` — The next evolution: sorted string tables replace the hash index with sorted on-disk structure, enabling range queries
- [general] `bitcask-compaction-atomicity` — The current compaction isn't atomic: a crash between deleting old segments and renaming the active segment could leave the store in an inconsistent state. Worth analyzing what guarantees are actually needed.
- [function] `log-structured-hash-table/bitcask.py:compact` — The active-segment rename dance (lines ~200-220) is subtle; trace through what happens to index entries pointing at the active segment during compaction

## Beliefs

- `bitcask-crc-covers-payload-only` — CRC32 is computed over `key_bytes + value_bytes`; the 12-byte header (which contains the CRC itself) is excluded from the checksum
- `bitcask-get-opens-fresh-handle` — `get()` opens a new file handle per call rather than using the `_file_handles` cache, meaning the handle cache is effectively unused for reads
- `bitcask-recovery-order-is-ascending` — Recovery scans segments in ascending ID order so that newer writes for the same key overwrite older index entries, and the highest-ID segment becomes active
- `bitcask-compaction-not-crash-safe` — Compaction performs delete-then-rename without journaling; a crash mid-compaction can leave the store in a state that `_recover()` cannot reconstruct correctly
- `bitcask-hint-files-exclude-tombstones` — `create_hint_files()` skips tombstone records and only emits entries whose index currently points to that segment+offset, so hint-based recovery cannot distinguish "key was deleted" from "key never existed in this segment"

