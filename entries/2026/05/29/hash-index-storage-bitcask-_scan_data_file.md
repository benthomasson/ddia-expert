# Function: _scan_data_file in hash-index-storage/bitcask.py

**Date:** 2026-05-29
**Time:** 06:48

## `_scan_data_file` — Rebuild the in-memory index by sequentially reading every record in a data file

### Purpose

This is the **crash recovery** path for a single data file. When a Bitcask store starts up and no hint file exists for a given data file, this method walks the file record-by-record to reconstruct the `keydir` — the in-memory hash map that maps each key to its on-disk location. Without this, the store would have no idea where any value lives.

It exists as the fallback to `_load_hint_file`, which does the same job faster using a precomputed summary. Hint files are written during compaction; freshly-written data files that were never compacted will always require a full scan.

### Contract

**Preconditions:**
- `file_id` corresponds to an existing `.data` file on disk
- The data file contains zero or more records in the Bitcask binary format: `[header | key_bytes | value_bytes]`
- Records are contiguous — no padding or alignment gaps between them
- The file is not currently being written to (this is called on immutable files or during startup before writes begin)

**Postconditions:**
- `self.keydir` reflects the latest state of every key in this file, **accounting for ordering**: later records in the file overwrite earlier ones, and tombstones (value_size == 0) remove the key entirely
- The reader's file position is at or past the end of the file

**Invariant:** Because `_rebuild_index` calls this on file IDs in sorted (ascending) order, a key written in file 3 will overwrite a key from file 1. This gives last-writer-wins semantics across the entire store.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `file_id` | `int` | Numeric identifier of the data file (maps to `{file_id}.data` on disk). Must correspond to an existing file. |

### Return Value

None. All output is via mutation of `self.keydir`.

### Algorithm

1. **Resolve the file path** and get a read handle via `_get_reader` (which caches open file descriptors in `self.file_handles`).
2. **Seek to position 0** — necessary because the reader may have been used before and left at an arbitrary position.
3. **Get the total file size** via `os.path.getsize` to know when to stop.
4. **Loop** while the current position is less than the file size:
   - Record the current `offset` — this is the byte position where the record starts.
   - Read `HEADER_SIZE` (16 bytes): a packed struct of `(timestamp: float64, key_size: uint32, value_size: uint32)`.
   - If we got a short read (truncated file), break — this handles files that were partially written before a crash.
   - Unpack the header to get `ts`, `key_size`, `val_size`.
   - Read `key_size` bytes for the key, then **read and discard** `val_size` bytes for the value. The value isn't needed for the index — only the location metadata matters.
   - Decode the key from UTF-8.
   - Compute `record_size = HEADER_SIZE + key_size + val_size`.
   - **Tombstone check:** if `val_size == 0`, this is a deletion marker — remove the key from `keydir` (using `pop` with a default to avoid `KeyError` if the key was never seen).
   - **Otherwise**, insert/overwrite the key in `keydir` with a `KeyEntry` pointing to this record's location.

### Side Effects

- **Mutates `self.keydir`** — adds, overwrites, or removes entries.
- **Mutates reader file position** — leaves it at or near EOF.
- **Opens a file handle** via `_get_reader` if one wasn't already cached for this `file_id`, which also mutates `self.file_handles`.

### Error Handling

Mostly absent — this method is **crash-tolerant but not error-defensive**:

- A **truncated header** (short read) silently breaks out of the loop. This handles the case where the process crashed mid-write.
- A **truncated key or value** (short read after the header) is *not* detected. If `key_size` claims 100 bytes but only 50 remain, `reader.read(key_size)` returns 50 bytes, `decode("utf-8")` may succeed or raise `UnicodeDecodeError`, and the index entry will point to a corrupted record.
- No **checksum validation** — there's no CRC in the record format, so bit-rot or partial writes that don't truncate the header will produce silently wrong data.
- `os.path.getsize` and `open` can raise `OSError`/`FileNotFoundError` — these propagate uncaught.

### Usage Patterns

Called exactly once per data file during startup, from `_rebuild_index`:

```python
def _rebuild_index(self, file_ids: list[int]):
    for fid in sorted(file_ids):
        if os.path.exists(self._hint_path(fid)):
            self._load_hint_file(fid)
        else:
            self._scan_data_file(fid)
```

The sorted order is critical — it ensures that if key `"foo"` appears in both file 1 and file 3, file 3's entry wins. The same scan logic is duplicated in `compact()` for collecting live records from immutable files.

### Dependencies

| Dependency | Usage |
|------------|-------|
| `struct.unpack` | Deserializes the binary record header |
| `os.path.getsize` | Determines file boundary for the scan loop |
| `HEADER_FORMAT` / `HEADER_SIZE` | Format string `<dII` (little-endian: double + 2 uint32s) = 16 bytes |
| `KeyEntry` | Dataclass holding `(file_id, offset, size, timestamp)` |
| `self._get_reader` | Returns a cached read-mode file handle |
| `self.keydir` | The in-memory hash index being rebuilt |

### Assumptions Not Enforced by Types

1. **Records are contiguous and correctly sized** — there is no framing or length-prefix at the file level. If a record's `key_size + val_size` doesn't match the actual bytes written, every subsequent record in the file will be misaligned and produce garbage.
2. **Keys are valid UTF-8** — `decode("utf-8")` will raise on invalid bytes, but nothing prevents binary keys from being written via `put()` if a caller bypasses the string type hint.
3. **`val_size == 0` is the sole tombstone signal** — a legitimate empty-string value is indistinguishable from a delete. This is consistent with `delete()` writing `""` and `get()` returning `None` for `""`, but it means the store cannot represent the empty string as a value.
4. **File IDs are scanned in ascending order** — correctness depends on the caller (`_rebuild_index`) sorting `file_ids`. Nothing in this method enforces or verifies ordering.
5. **No concurrent writers** — the method reads `file_size` once upfront. If another process appends to the file during the scan, those records are silently skipped.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:_load_hint_file` — The fast-path alternative that avoids scanning the data file; compare what metadata it stores vs. what `_scan_data_file` recomputes
- [function] `hash-index-storage/bitcask.py:compact` — Duplicates the scan logic for collecting live records; understanding compaction explains why hint files exist
- [general] `bitcask-paper-design` — The original Bitcask paper (Riak) describes the keydir, hint files, and merge process that this code implements — reading it clarifies why the on-disk format has no checksums or length-prefix framing
- [file] `hash-index-storage/test_bitcask.py` — Test cases reveal edge cases the implementation handles (or doesn't): crash recovery, tombstone semantics, compaction correctness
- [general] `ddia-chapter3-hash-indexes` — DDIA Chapter 3's hash index section motivates this entire design and discusses the tradeoffs vs. SSTables/LSM trees

## Beliefs

- `bitcask-tombstone-is-empty-value` — A tombstone is represented as a record with `val_size == 0`; the store cannot distinguish a deleted key from one with an empty string value
- `bitcask-scan-order-determines-correctness` — `_scan_data_file` relies on being called in ascending file ID order so that newer writes overwrite older ones in the keydir
- `bitcask-no-checksum-validation` — The record format contains no CRC or checksum; corrupted records that preserve valid header lengths are silently accepted during index rebuild
- `bitcask-truncated-record-silent-break` — A short header read causes the scan to stop silently, tolerating crash-truncated files, but a short key/value read after a valid header is not detected and may corrupt the index

