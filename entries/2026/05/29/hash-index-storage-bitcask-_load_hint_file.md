# Function: _load_hint_file in hash-index-storage/bitcask.py

**Date:** 2026-05-29
**Time:** 06:49

## `_load_hint_file` — Fast Index Recovery from Hint Files

### Purpose

`_load_hint_file` reconstructs the in-memory key index (`self.keydir`) from a precomputed hint file instead of scanning the full data file record-by-record. Hint files are a Bitcask optimization: during compaction, a compact summary of each record's location metadata is written alongside the merged data file. On startup, loading a hint file is O(index entries) with minimal I/O, whereas `_scan_data_file` must read and skip over every value payload — potentially gigabytes of data — just to extract the same key-to-location mappings.

### Contract

- **Precondition**: A hint file at `{data_dir}/{file_id}.hint` must exist and be well-formed. The caller (`_rebuild_index`) checks `os.path.exists` before calling.
- **Postcondition**: Every key described in the hint file has a corresponding `KeyEntry` in `self.keydir`. If the same key appears multiple times in the hint file, the last entry wins (simple overwrite).
- **Invariant**: After loading, `self.keydir[key]` points to a valid `(file_id, offset, size)` tuple that can be used by `_read_record` to retrieve the full record from the data file.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `self` | `BitcaskStore` | The store instance whose `keydir` will be populated |
| `file_id` | `int` | Numeric ID of the data file whose hint file to load. Maps to `{file_id}.hint` on disk. |

### Return Value

None. The method mutates `self.keydir` as a side effect.

### Algorithm

1. **Bulk read**: Opens the hint file and reads the entire contents into memory as a single `bytes` object. This is an intentional choice — hint files are small (they contain no value data), so buffering the whole thing avoids repeated syscalls.

2. **Iterate fixed-header records**: Walks through the byte buffer using a manual position cursor `pos`:

   a. **Unpack the fixed header** (24 bytes, `HINT_FORMAT = "<IQId"`):
      - `fid` (uint32) — the data file ID containing this record
      - `offset` (uint64) — byte offset within that data file
      - `size` (uint32) — total record size in the data file (header + key + value)
      - `ts` (double) — timestamp when the record was written

   b. **Read the variable-length key**: Unpacks a 4-byte little-endian uint32 for `key_size`, then reads that many bytes and decodes as UTF-8.

   c. **Insert into keydir**: Creates a `KeyEntry(fid, offset, size, ts)` and stores it under the decoded key. If the key already exists (from an earlier file's hint), it is silently overwritten — this is correct because `_rebuild_index` processes files in sorted ID order, so later files contain newer data.

### Side Effects

- **Mutates `self.keydir`**: Adds or overwrites entries for every key found in the hint file.
- **File I/O**: Opens and reads the hint file. The file handle is closed immediately via the `with` block.
- **No reader handle cached**: Unlike `_scan_data_file` which uses `_get_reader`, this method does not register a file handle in `self.file_handles` — hint files are read-once at startup and never accessed again.

### Error Handling

None. The method will raise:
- `FileNotFoundError` if the hint file doesn't exist (caller is responsible for checking).
- `struct.error` if the file is truncated or corrupt mid-record.
- `UnicodeDecodeError` if a key contains invalid UTF-8 bytes.
- `IndexError` / `struct.error` if `key_size` extends beyond the buffer.

No corruption detection (checksums, magic bytes) is performed. The method trusts that the hint file is well-formed.

### Usage Patterns

Called exclusively by `_rebuild_index` during store initialization:

```python
def _rebuild_index(self, file_ids):
    for fid in sorted(file_ids):
        if os.path.exists(self._hint_path(fid)):
            self._load_hint_file(fid)    # fast path
        else:
            self._scan_data_file(fid)    # slow path
```

The caller obligation is simple: only call this when a `.hint` file is known to exist. The symmetric writer is `_write_hint_file`, called during compaction to produce the hint files that this method later consumes.

### Dependencies

- `struct` — binary packing/unpacking with little-endian format strings
- `HINT_FORMAT` / `HINT_HEADER_SIZE` — module-level constants defining the fixed portion of each hint record (`"<IQId"`, 24 bytes)
- `KeyEntry` dataclass — the value type stored in `keydir`
- `self._hint_path(file_id)` — resolves to `{data_dir}/{file_id}.hint`

### Notable Assumptions

1. **Hint file format includes a key-size field not covered by `HINT_FORMAT`**: The 4-byte `key_size` is packed separately from the fixed header. This means each hint record is actually `HINT_HEADER_SIZE + 4 + len(key_bytes)` — a variable-length format with a two-part header. This asymmetry between the format constant and actual layout is easy to miss.

2. **No tombstone handling**: Unlike `_scan_data_file` which checks `val_size == 0` to identify tombstones and removes keys from `keydir`, hint files are only produced during compaction — and compaction already filters out tombstones. So hint files are assumed to contain only live records.

3. **`fid` in the hint record may differ from the `file_id` parameter**: The hint record stores its own `fid` field (the data file ID where the record lives). In practice after compaction these should match, but the code trusts the embedded `fid` over the parameter — which is the correct behavior if a hint file ever mapped across data files.

4. **UTF-8 keys only**: Keys are decoded with `.decode("utf-8")` with no error handling. Binary keys or other encodings will raise.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:_write_hint_file` — The symmetric writer; understanding the write format clarifies why the read parses two separate headers
- [function] `hash-index-storage/bitcask.py:_scan_data_file` — The slow-path alternative; compare to understand what hint files save you (tombstone handling, value skipping)
- [function] `hash-index-storage/bitcask.py:compact` — The only producer of hint files; shows when and why they're created
- [general] `bitcask-hint-file-format` — The original Bitcask paper's hint file design and how this implementation diverges (e.g., no CRC, variable-length key encoding)
- [file] `hash-index-storage/test_bitcask.py` — Tests covering compaction and restart to verify hint file round-tripping

---

## Beliefs

- `hint-files-only-from-compaction` — Hint files are only written during `compact()`; the normal write path (`put`/`delete`) never produces them, so un-compacted data files always use the slow `_scan_data_file` recovery path
- `hint-no-tombstones` — Hint files contain no tombstone records because compaction filters them out before writing; `_load_hint_file` therefore never removes keys from `keydir`
- `hint-format-variable-length` — Each hint record is `HINT_HEADER_SIZE + 4 + key_length` bytes; the `HINT_FORMAT` constant covers only the first 24 bytes, not the full record
- `hint-no-integrity-check` — Hint files have no checksums, magic numbers, or version fields; a corrupt or truncated hint file will cause struct unpacking errors rather than graceful recovery
- `rebuild-order-matters` — `_rebuild_index` processes file IDs in sorted order so that newer records overwrite older ones in `keydir`; `_load_hint_file` relies on this by unconditionally overwriting existing entries

