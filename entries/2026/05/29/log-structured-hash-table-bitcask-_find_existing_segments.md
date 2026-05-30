# Function: _find_existing_segments in log-structured-hash-table/bitcask.py

**Date:** 2026-05-29
**Time:** 12:22

## `_find_existing_segments`

### Purpose

This is the segment discovery method for the Bitcask store. It scans the data directory to find all segment files on disk and returns them in order. It exists because the store needs to know which segments exist — at startup for recovery, during compaction to identify frozen segments, and for status reporting. The store doesn't maintain a persistent manifest of segments; instead, the filesystem *is* the manifest, and this method reads it on demand.

### Contract

- **Precondition**: `self._dir` must be a valid, readable directory. This is guaranteed by the constructor's `os.makedirs(directory, exist_ok=True)` call.
- **Postcondition**: Returns segments sorted ascending by segment ID (the default `sort()` on tuples compares by first element). The returned paths are absolute (joined with `self._dir`).
- **Invariant**: Every returned entry corresponds to a file that actually exists on disk at the time of the call. The method does not create or modify any files.

### Parameters

None (beyond `self`). It reads `self._dir` to know where to look.

### Return Value

`list[tuple[int, str]]` — a list of `(segment_id, absolute_path)` pairs, sorted by segment ID ascending. Returns an empty list if no segment files exist. The caller must handle the empty case — `_recover` does this by opening a fresh segment.

### Algorithm

1. List all entries in `self._dir` via `os.listdir`.
2. Filter to filenames matching the pattern `segment_NNNNNN.dat` — specifically, those starting with `"segment_"` and ending with `".dat"`.
3. Extract the segment ID by slicing out the numeric portion between the prefix and suffix, converting to `int`. For a file named `segment_000042.dat`, this yields `42`.
4. Build a full path by joining with `self._dir`.
5. Sort the list. Since tuples sort lexicographically and segment ID is the first element, this produces ascending ID order.
6. Return the sorted list.

### Side Effects

None. This is a pure read from the filesystem — no mutations to `self`, no file creation, no I/O beyond the directory listing.

### Error Handling

No explicit error handling. Several implicit failure modes:

- **`os.listdir` fails**: Raises `OSError`/`FileNotFoundError` if the directory was removed after construction. Not caught here.
- **`int()` conversion fails**: If a file matches the prefix/suffix pattern but has non-numeric content between them (e.g., `segment_abc.dat`), `int()` raises `ValueError`. This would crash the method — the code assumes all matching filenames are well-formed.
- **Race condition**: A segment file could be deleted between `listdir` and a subsequent read by the caller. The method doesn't guard against this; callers assume the returned paths are valid.

### Usage Patterns

Called in five contexts throughout the class:

1. **`_recover`** — at startup, to rebuild the index from all segments on disk.
2. **`_frozen_segment_paths`** — to find non-active segments eligible for compaction.
3. **`segments`** — to report segment metadata to callers.
4. **`total_disk_usage`** / **`num_segments`** — properties that enumerate segments for stats.
5. **`rebuild_index`** — full index reconstruction on demand.

Callers rely on the sort order. `_recover` depends on it to process oldest segments first (so newer writes overwrite older index entries), and to pick the highest-ID segment as the active one.

### Dependencies

- `os.listdir` and `os.path.join` from the standard library.
- Implicitly depends on the naming convention enforced by `_segment_path`, which produces `segment_{id:06d}.dat`. The parsing here must stay in sync with that format — if the naming scheme changes in one place but not the other, segment discovery breaks silently.

### Untyped Assumptions

- Filenames matching the pattern are always created by this class. No external process will drop a `segment_xyz.dat` into the directory.
- The zero-padded format (`%06d`) means `int()` correctly handles leading zeros, but the code would also work with non-padded names since `int("000042") == 42`.
- The sort is by integer ID, not lexicographic filename order. These happen to agree because of the zero-padding, but the code correctly sorts on the parsed integer, making it robust to non-padded names.

## Topics to Explore

- [function] `log-structured-hash-table/bitcask.py:_recover` — The primary consumer; understanding recovery shows why sort order and completeness matter here
- [function] `log-structured-hash-table/bitcask.py:compact` — Compaction creates and deletes segments, making it the method most likely to introduce race conditions with `_find_existing_segments`
- [function] `log-structured-hash-table/bitcask.py:_segment_path` — The naming convention that this method's parsing must mirror; changing one without the other is a latent bug
- [general] `bitcask-paper-segment-management` — How the original Bitcask paper describes segment file management and whether hint files factor into discovery
- [file] `log-structured-hash-table/test_bitcask.py` — Test coverage for segment discovery, especially edge cases like empty directories and corrupted filenames

## Beliefs

- `bitcask-segment-discovery-is-filesystem-based` — `_find_existing_segments` uses `os.listdir` as the source of truth for which segments exist; there is no persistent manifest file
- `bitcask-segment-sort-order-is-by-id` — Segments are sorted by parsed integer ID, not lexicographic filename, ensuring oldest-first processing during recovery
- `bitcask-segment-naming-must-sync` — The filename parsing in `_find_existing_segments` (prefix/suffix slicing) must stay in sync with the format string in `_segment_path`; no shared constant enforces this
- `bitcask-no-validation-on-segment-filenames` — If a file matching `segment_*.dat` contains non-numeric characters in the ID position, `_find_existing_segments` will crash with `ValueError`

