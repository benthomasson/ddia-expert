# Function: compact in log-structured-hash-table/bitcask.py

**Date:** 2026-05-28
**Time:** 18:57

## `BitcaskStore.compact` — Log Segment Compaction

### Purpose

`compact` is the garbage collection mechanism for this Bitcask store. Because the store is append-only, updates to a key don't overwrite the old value — they write a new record to the active segment. Deletions write a tombstone rather than removing anything. Over time, frozen (closed) segments accumulate stale records: superseded values and tombstoned keys. `compact` merges all frozen segments into a single new segment containing only the live records, then deletes the old segments to reclaim disk space.

This is the operation described in DDIA Chapter 3 as "compaction" for hash-indexed log-structured storage — the defining maintenance operation that makes append-only writes practical.

### Contract

**Preconditions:**
- The store must be open (`_active_file` is valid and writable).
- No concurrent writers — there is no locking; this is single-threaded by assumption.

**Postconditions:**
- All frozen segments are replaced by at most one compacted segment.
- The in-memory index (`_index`) is updated so every key points to its new location.
- The active segment remains writable with the highest segment ID.
- Stale file handles are closed and removed from `_file_handles`.

**Invariant preserved:** The active segment always has the highest segment ID on disk. This is critical for recovery (`_recover` assumes the last segment is active).

### Parameters

None beyond `self`. The method operates on internal state.

### Return Value

Returns `int` — the number of stale records removed (total records scanned minus live records written). A return of `0` with no frozen segments means there was nothing to compact. A return of `0` with frozen segments would mean every record in every frozen segment was still live (unlikely but possible).

### Algorithm

The method runs in five phases:

**Phase 1 — Scan frozen segments to find live entries:**
Iterates every record in every frozen segment (oldest to newest). Builds a `live_entries` dict mapping each key to its most recent `(segment_path, offset)`. Tombstones remove keys from `live_entries`. The oldest-to-newest ordering matters: later writes for the same key overwrite earlier ones, so the final state of `live_entries` reflects the most recent value per key across all frozen segments.

**Phase 2 — Filter against the authoritative index:**
Not every entry in `live_entries` is truly live. A key might have been overwritten *in the active segment* after the frozen segment was closed. The filter `self._index[key] == (seg_path, offset)` checks whether the in-memory index still points to this frozen-segment record. If the index points elsewhere (e.g., the active segment), the frozen record is stale and excluded.

**Phase 3 — Write compacted segment:**
Closes the active file, bumps the segment counter, and opens a new compacted segment file. For each entry to keep, it re-reads the value from the original segment, recomputes the CRC, and writes a fresh record to the compacted segment. It builds `new_index_entries` with the new offsets.

**Phase 4 — Clean up old segments:**
Updates `_index` with the new locations, closes cached read handles for old segments, and deletes the old frozen segment files along with any associated hint files.

**Phase 5 — Restore the active segment with the highest ID:**
Bumps the segment counter again, renames the old active segment file to the new highest ID, updates all index entries that pointed to the old active path, and reopens the active file for appending. This ensures the active segment has a higher ID than the compacted segment, preserving the recovery invariant.

### Side Effects

This method is heavy on side effects:

- **Disk I/O**: Reads all frozen segments, writes one new segment, deletes old segments and hint files, renames the active segment file.
- **Index mutation**: `_index` is updated in-place with new file paths and offsets.
- **File handle management**: Closes `_active_file`, closes cached handles in `_file_handles`, reopens active file.
- **Counter mutation**: `_segment_counter` is incremented twice (once for the compacted segment, once for the renamed active).
- **Active segment rename**: The active segment is renamed on disk and its path changes in memory.

### Error Handling

There is essentially none. If any `open`, `read`, `write`, `os.remove`, or `os.rename` call fails, the exception propagates uncaught. This leaves the store in a partially compacted state:

- If the crash happens after writing the compacted segment but before deleting old segments, recovery will scan both old and new segments — the index will be correct (latest write wins) but disk space won't be reclaimed.
- If the crash happens after deleting old segments but before renaming the active segment, the active segment path in memory won't match disk. Recovery from scratch would fix this, but the in-process instance would be broken.

The CRC in the compacted segment is recomputed fresh, so compaction doesn't propagate corrupted data — but it also doesn't *verify* the CRC of records it reads from old segments during compaction (unlike `_scan_segment`, which does).

### Usage Patterns

Called in two ways:

1. **Automatically** from `put()` when the number of frozen segments reaches `_auto_compact_threshold` (default 5).
2. **Manually** by the caller for explicit maintenance.

Caller obligations: don't call `compact` concurrently with reads or writes. The method temporarily closes the active file, so any interleaved `put` or `get` during compaction would fail or corrupt state.

### Dependencies

- `struct` — binary record format packing/unpacking
- `zlib.crc32` — CRC32 checksums for data integrity
- `os` — file deletion (`remove`), renaming (`rename`), existence checks (`path.exists`)
- Internal: `_frozen_segment_paths`, `_segment_path`, `_hint_path`, `_index`, `_file_handles`, `_active_file`, `_active_path`, `_segment_counter`

### Assumptions Not Enforced by Types

1. **Single-threaded access** — no locks protect `_index`, `_active_file`, or the segment counter during the multi-step mutation sequence.
2. **Segment ordering** — the scan assumes frozen segments are returned in creation order so that later writes correctly shadow earlier ones. `_frozen_segment_paths` delegates to `_find_existing_segments` which sorts by ID, so this holds as long as IDs increase monotonically.
3. **No external modification** — the method assumes no other process modifies segment files on disk between the scan and the delete.
4. **Active segment not included** — `_frozen_segment_paths` excludes the active path by identity comparison (`path != self._active_path`). If the active path string doesn't exactly match what's on disk (e.g., symlinks, relative vs absolute), frozen segments could be misidentified.
5. **CRC not verified on read during compaction** — unlike `_scan_segment`, the compaction scan doesn't check CRCs, so a corrupted record in a frozen segment could be silently copied into the compacted segment.

---

## Topics to Explore

- [function] `log-structured-hash-table/bitcask.py:_recover` — Rebuild logic on startup; understanding recovery explains why the segment-ID ordering invariant matters
- [function] `log-structured-hash-table/bitcask.py:put` — Shows auto-compaction triggering and segment rotation, the two callers of `compact`
- [file] `log-structured-hash-table/test_bitcask.py` — Test cases revealing expected compaction behavior and edge cases
- [general] `bitcask-crash-safety` — The compaction sequence is not atomic; worth analyzing which failure points leave recoverable vs corrupted state
- [function] `log-structured-hash-table/bitcask.py:create_hint_files` — Hint files accelerate recovery but are deleted during compaction and not regenerated; understand the tradeoff

## Beliefs

- `compact-skips-crc-validation` — Compaction reads records from frozen segments without verifying CRC checksums, unlike `_scan_segment` which stops at the first CRC mismatch
- `compact-preserves-active-highest-id` — After compaction, the active segment always has the highest segment ID on disk, which is required for `_recover` to identify the active segment
- `compact-not-crash-safe` — The compaction sequence (write new, update index, delete old, rename active) is not atomic; a crash mid-compaction can leave orphaned segments or a misnamed active file
- `compact-filters-against-live-index` — Records in frozen segments are only kept if `_index` still points to that exact `(path, offset)`, correctly excluding keys overwritten in the active segment
- `compact-single-threaded-assumption` — No synchronization protects the index, file handles, or segment counter during compaction; concurrent access would corrupt state

