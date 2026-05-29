# Function: _write_hint_file in hash-index-storage/bitcask.py

**Date:** 2026-05-28
**Time:** 18:58

## `_write_hint_file`

### Purpose

`_write_hint_file` serializes a set of index entries to a companion `.hint` file alongside a `.data` file. Hint files exist as a startup optimization: instead of scanning every record in a data file to rebuild the in-memory key directory (`keydir`), the engine can read the hint file, which contains only the index metadata (no values), making recovery dramatically faster — O(keys) small reads instead of O(data-size) full scans.

This is a core Bitcask design concept from the original Riak paper. Hint files are only written during **compaction**, never during normal writes.

### Contract

- **Precondition**: `entries` contains the final, deduplicated set of live key→location mappings for `file_id`. The corresponding `.data` file must already be fully written and flushed — the offsets and sizes in each `KeyEntry` must be valid positions in that data file.
- **Postcondition**: A binary file at `{file_id}.hint` exists on disk containing a serialized representation of every entry, readable by `_load_hint_file`.
- **Invariant**: The hint file is a pure projection of the data file's index — it contains no values, only enough metadata to reconstruct `keydir` entries without reading the data file.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `file_id` | `int` | Numeric identifier for the data file this hint corresponds to. Used to derive the path `{data_dir}/{file_id}.hint`. |
| `entries` | `list[tuple[str, KeyEntry]]` | List of `(key_string, index_entry)` pairs. Each pair maps a key to its location in the data file. Order doesn't matter semantically but is preserved in the file. An empty list produces an empty hint file. |

**Edge cases**: If `entries` is empty, the method creates a valid but zero-length hint file. Keys containing non-ASCII characters will be encoded as UTF-8, and the key length field reflects the byte length, not the character count.

### Return Value

None. The method operates purely through side effects.

### Algorithm

For each `(key, entry)` pair, three writes are appended sequentially:

1. **Index header** (24 bytes) — `struct.pack(HINT_FORMAT, ...)` writes four fields in little-endian format:
   - `file_id` as `uint32` (I) — which data file holds this record
   - `offset` as `uint64` (Q) — byte offset within that data file
   - `size` as `uint32` (I) — total record size (header + key + value)
   - `timestamp` as `float64` (d) — original write timestamp

2. **Key length** (4 bytes) — `struct.pack("<I", len(key_bytes))` writes the UTF-8 byte length of the key as a little-endian `uint32`.

3. **Key data** (variable) — the raw UTF-8 bytes of the key.

The resulting per-entry layout is:

```
[file_id:4][offset:8][size:4][timestamp:8] [key_len:4] [key_bytes:variable]
 ──────────── 24 bytes ────────────────── ─── 4 ───── ─── key_len ────────
```

### Side Effects

- **File I/O**: Creates or overwrites `{data_dir}/{file_id}.hint` in binary mode (`"wb"`). The file is written without explicit `fsync` — contrast with `_write_record`, which does fsync when `sync_writes` is enabled. This is acceptable because hint files are a rebuild optimization, not a durability mechanism; if lost, the engine falls back to scanning the data file.
- **No state mutation**: Does not modify `keydir`, `file_handles`, or any other instance state.

### Error Handling

No explicit error handling. The method will propagate:
- `OSError` / `PermissionError` if the hint file can't be created or written
- `struct.error` if any `KeyEntry` field overflows its format bounds (e.g., `offset` exceeding `uint64` range, though practically impossible)
- `UnicodeEncodeError` if a key contains characters that can't be UTF-8 encoded (impossible for valid Python `str`)

The `with` block ensures the file handle is closed even on exception, but a partial write will leave a corrupted hint file on disk. Since `_load_hint_file` doesn't validate checksums, a truncated hint file would cause an `struct.error` or `IndexError` on the next startup.

### Usage Patterns

Called exclusively from `compact()`, in two places:

1. **Mid-compaction rotation** — when a merged data file reaches `max_file_size`, the hint file for the just-completed segment is written before starting a new segment.
2. **End of compaction** — after the final merged data file is closed.

Callers are responsible for collecting the `(key, KeyEntry)` pairs as records are written to the merged file. The entries must reflect the *new* file's offsets, not the original file's.

### Dependencies

- `struct` — binary packing via format string `HINT_FORMAT = "<IQId"` (24 bytes per entry header)
- `self._hint_path(file_id)` — resolves to `{data_dir}/{file_id}.hint`
- `KeyEntry` dataclass — provides `file_id`, `offset`, `size`, `timestamp` fields

The format must stay in sync with `_load_hint_file`, which reads the same binary layout. Any change to `HINT_FORMAT` or the key-length encoding breaks backward compatibility.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:_load_hint_file` — The mirror-image deserializer; understanding the read side reveals what assumptions the hint format makes about alignment and completeness
- [function] `hash-index-storage/bitcask.py:compact` — The only caller; shows how hint entries are accumulated during the merge-write loop and when rotation triggers a mid-compaction hint flush
- [function] `hash-index-storage/bitcask.py:_scan_data_file` — The slow-path fallback when no hint file exists; comparing its cost to `_load_hint_file` quantifies the optimization hint files provide
- [general] `bitcask-hint-durability` — The hint file is written without `fsync` and has no checksums — explore what happens on crash during compaction and whether a partial hint file could silently lose keys on restart
- [general] `bitcask-paper-design` — The original Bitcask paper by Riak describes hint files, merge semantics, and the keydir; compare this implementation's choices against the paper's recommendations

## Beliefs

- `hint-file-no-fsync` — Hint files are written without `fsync`, making them a best-effort optimization that can be lost on crash without data loss (the data file remains authoritative)
- `hint-format-symmetry` — `_write_hint_file` and `_load_hint_file` must use identical binary layouts; the format is `[HINT_FORMAT header (24 bytes)][key_len uint32 (4 bytes)][key_bytes (variable)]` per entry
- `hint-written-only-during-compaction` — `_write_hint_file` is called exclusively from `compact()`, never during normal `put`/`delete` operations
- `hint-file-no-integrity-check` — Neither writer nor reader performs checksum validation; a truncated or corrupted hint file will cause a deserialization error on startup rather than a graceful fallback to data file scanning

