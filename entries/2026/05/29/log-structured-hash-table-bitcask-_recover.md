# Function: _recover in log-structured-hash-table/bitcask.py

**Date:** 2026-05-29
**Time:** 06:44

## `BitcaskStore._recover`

### Purpose

`_recover` rebuilds the in-memory hash index (`self._index`) from on-disk segment files when a `BitcaskStore` is instantiated. This is the crash-recovery path: since the index lives only in memory, it must be reconstructed from the durable log every time the store opens. Without it, the store would start empty regardless of what data was previously written.

### Contract

- **Precondition**: `self._dir` exists on disk. `self._index` is empty. `self._active_file` is `None`. Called exactly once, at the end of `__init__`.
- **Postcondition**: `self._index` maps every live key to its `(segment_path, byte_offset)`. `self._active_file` is open for appending. `self._segment_counter` reflects the highest segment ID on disk (or 0 if no segments existed).
- **Invariant**: Segments are replayed oldest-to-newest, so newer writes for the same key overwrite older index entries — the last writer wins, matching the append-only semantics.

### Parameters

None (operates on `self`).

### Return Value

None. All effects are mutations to instance state.

### Algorithm

1. **Discover segments** — calls `_find_existing_segments()`, which scans the directory for files matching `segment_NNNNNN.dat` and returns them sorted by ID (ascending = oldest first).

2. **Empty-directory fast path** — if no segments exist, this is a fresh store. Sets the counter to 0, opens a new empty segment via `_open_new_segment()`, and returns.

3. **Replay loop** — iterates segments oldest-to-newest. For each segment:
   - If a `.hint` file exists alongside it, loads the hint file (`_load_hint_file`). Hint files contain only `(key, offset)` pairs — a compact index that skips full record parsing.
   - Otherwise, falls back to `_scan_segment`, which reads every record, verifies CRC32, and updates the index. Tombstone records (`TOMBSTONE` sentinel value) cause the key to be *removed* from the index.

4. **Activate the newest segment** — the segment with the highest ID becomes the active (writable) segment. Sets `_segment_counter` to that ID, records its path, and opens it in append-binary mode (`"ab"`).

### Side Effects

| Effect | Detail |
|--------|--------|
| `self._index` | Populated with all live key→(path, offset) mappings |
| `self._segment_counter` | Set to max segment ID or 0 |
| `self._active_path` | Set to the newest segment's path |
| `self._active_file` | Opened for appending to the newest segment |
| Disk I/O | Reads every segment (and hint) file in the directory |

### Error Handling

There is essentially none — this method lets exceptions propagate:

- `OSError` / `FileNotFoundError` from `open()` or `os.listdir()` will crash the constructor. There is no retry or fallback.
- `_scan_segment` silently stops scanning when it encounters a partial write or CRC mismatch (it `break`s), treating trailing corruption as end-of-segment. This is intentional: partial writes from a crash are discarded.
- `_load_hint_file` similarly stops on short reads but does **no CRC verification** — it trusts hint files unconditionally. A corrupted hint file will silently load wrong offsets into the index.

### Usage Patterns

Called exactly once, from `__init__`. Not intended for external use (prefixed with `_`). If you need to rebuild the index after the store is already open, use `rebuild_index()` instead — though note that `rebuild_index` does not reset `_active_file` or `_segment_counter`.

```python
store = BitcaskStore("/tmp/mydb")  # _recover() runs here
```

### Dependencies

| Dependency | Purpose |
|------------|---------|
| `_find_existing_segments()` | Directory scan, sorted segment discovery |
| `_load_hint_file(hint_path, seg_path)` | Fast index load from precomputed hint |
| `_scan_segment(seg_path)` | Full segment scan with CRC verification |
| `_open_new_segment()` | Create and open a fresh segment file |
| `_hint_path(seg_path)` | Derive `.hint` path from `.dat` path |
| `os.path.exists` | Check for hint file presence |

### Assumptions Not Enforced by Types

1. **Segment filenames are well-formed** — `_find_existing_segments` does `int(fname[len("segment_"):-len(".dat")])`. Any file matching the prefix/suffix pattern but with a non-numeric middle (e.g., `segment_00foo0.dat`) will raise `ValueError`.
2. **Hint files are consistent with their segments** — `_load_hint_file` loads offsets without verifying the referenced records exist or match. If a hint file is stale (from a previous compaction) but its `.dat` was replaced, the index will contain garbage offsets.
3. **No concurrent writers** — there is no file locking. Two processes opening the same directory will both append to the same active segment, corrupting it.
4. **The newest segment is always the right one to append to** — if a compaction was interrupted mid-way (after deleting old segments but before renaming the active), recovery could pick a compacted segment as active.

---

## Topics to Explore

- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — The full-scan fallback: how it handles CRC checks, tombstones, and partial writes at segment boundaries
- [function] `log-structured-hash-table/bitcask.py:compact` — Compaction rewrites frozen segments and renumbers the active segment; understanding it clarifies why recovery always picks the highest-ID segment
- [function] `log-structured-hash-table/bitcask.py:_load_hint_file` — Hint files skip CRC verification entirely; worth understanding the trade-off between startup speed and integrity
- [general] `bitcask-crash-recovery-guarantees` — What data loss scenarios are possible given that this implementation has no write-ahead log and no fsync calls
- [file] `log-structured-hash-table/test_bitcask.py` — Tests for recovery behavior: what crash scenarios are covered and which edge cases (corrupted hints, interrupted compaction) are not

---

## Beliefs

- `recover-replay-order` — `_recover` replays segments oldest-to-newest so that the latest write for a key wins, matching Bitcask's last-writer-wins semantics
- `hint-files-skip-crc` — `_load_hint_file` trusts hint file contents without CRC verification; a corrupted hint file will silently produce wrong index entries
- `recover-discards-trailing-corruption` — `_scan_segment` stops at the first CRC mismatch or short read, silently discarding any trailing partial writes rather than raising an error
- `no-concurrent-writer-protection` — `_recover` (and the store generally) assumes single-writer access; no file locking prevents two processes from corrupting the active segment
- `active-segment-is-highest-id` — After recovery, the segment with the numerically highest ID is always opened as the active writable segment

