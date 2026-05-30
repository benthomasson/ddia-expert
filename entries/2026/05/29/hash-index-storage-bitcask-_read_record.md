# Function: _read_record in hash-index-storage/bitcask.py

**Date:** 2026-05-29
**Time:** 11:58

## `_read_record` — Point-read a single record from a Bitcask data file

### Purpose

`_read_record` is the read-path counterpart to `_write_record`. Given a file ID, byte offset, and record size, it deserializes a single key-value record from disk. This is the only way the store retrieves values — the in-memory `keydir` hash index stores the location coordinates, and `_read_record` does the actual I/O to fetch the data.

It exists because Bitcask separates the index (in-memory hash map of key → location) from the data (append-only files on disk). Reads are always single-seek operations: look up the location in `keydir`, then call this method to fetch the bytes.

### Contract

**Preconditions:**
- `file_id` must correspond to an existing `.data` file in `self.data_dir`
- `offset` and `size` must describe a valid, complete record written by `_write_record` — the method trusts these values and does no bounds checking
- The record at that location must use the same `HEADER_FORMAT` (`<dII` — 16 bytes: double timestamp, two uint32 lengths)

**Postconditions:**
- Returns the decoded key, value, and original write timestamp
- The file handle's seek position is advanced to `offset + size`

**Invariant:** The returned key and value are the exact strings that were passed to `_write_record` when the record was created.

### Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `file_id` | `int` | Identifies which `.data` file to read from (maps to `{file_id}.data` on disk) |
| `offset` | `int` | Byte offset within that file where the record starts |
| `size` | `int` | Total byte length of the record (header + key bytes + value bytes) |

All three values come from a `KeyEntry` in `self.keydir`. The caller never computes these — they were captured at write time by `_write_record`.

### Return Value

Returns `(key, value, timestamp)` where:
- `key: str` — the decoded key
- `value: str` — the decoded value (may be `""` for tombstone records)
- `timestamp: float` — the `time.time()` value captured when the record was written

The caller (`get()`) is responsible for checking whether `value == ""` indicates a tombstone/deletion.

### Algorithm

1. **Obtain a read handle** via `_get_reader(file_id)`, which lazily opens the file in `"rb"` mode and caches the handle.
2. **Seek** to the exact byte offset where the record begins.
3. **Bulk-read** the entire record (`size` bytes) into memory in a single I/O call — this is important because it avoids multiple small reads.
4. **Unpack the header** — the first 16 bytes (`HEADER_SIZE`) are parsed as `(timestamp, key_size, val_size)` using the little-endian format `<dII`.
5. **Slice and decode the key** — bytes `[16 : 16 + key_size]`, decoded as UTF-8.
6. **Slice and decode the value** — bytes `[16 + key_size : 16 + key_size + val_size]`, decoded as UTF-8.
7. **Return** the triple.

The key insight: `size` is used only for the bulk read. The actual field boundaries come from `key_size` and `val_size` unpacked from the header. The `size` parameter is redundant with `HEADER_SIZE + key_size + val_size` — it's passed for convenience so the method can do a single `read()` call without first reading just the header.

### Side Effects

- **File I/O**: Seeks and reads from a file handle. The seek mutates the handle's position, which means concurrent reads on the same handle would race. This is safe in the current single-threaded design but would break under concurrent access.
- **Lazy file open**: `_get_reader` may open a new file handle and cache it in `self.file_handles` if this file hasn't been read before.

### Error Handling

There is essentially none — the method assumes valid inputs:

- If `file_id` doesn't exist on disk, `_get_reader` raises `FileNotFoundError`.
- If `offset`/`size` are wrong, `reader.read(size)` returns fewer bytes than expected, and `struct.unpack` raises `struct.error` on a short buffer.
- If the data is corrupted (e.g., `key_size` exceeds actual data), the slice will silently return truncated or garbage bytes. There is no checksum or CRC validation.
- If the key/value bytes aren't valid UTF-8, `.decode("utf-8")` raises `UnicodeDecodeError`.

### Usage Patterns

Called exclusively by `get()`:

```python
def get(self, key):
    entry = self.keydir.get(key)
    if entry is None:
        return None
    read_key, value, _ = self._read_record(entry.file_id, entry.offset, entry.size)
    assert read_key == key  # sanity check
```

The `assert` in `get()` acts as a weak integrity check — if the keydir is out of sync with the data files, the key won't match. But this is a development-time guard, not a production safety net (assertions can be disabled with `-O`).

Also used implicitly during compaction, where old records are re-read via direct file I/O rather than through this method (compaction opens its own temporary readers).

### Dependencies

- `struct` — for binary deserialization of the fixed-size header
- `_get_reader()` — manages the file handle cache (`self.file_handles`)
- Module-level constants `HEADER_FORMAT` (`"<dII"`) and `HEADER_SIZE` (16 bytes) — define the on-disk binary layout

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:_write_record` — The write-side counterpart; understanding the record format it produces is essential to understanding what `_read_record` deserializes
- [function] `hash-index-storage/bitcask.py:_scan_data_file` — An alternative read path that sequentially scans an entire file without knowing offsets in advance, used during index rebuild
- [function] `hash-index-storage/bitcask.py:compact` — Compaction re-reads records from immutable files but bypasses `_read_record`, opening its own file handles; worth understanding why
- [general] `bitcask-concurrency-safety` — The shared mutable file handles mean `_read_record` is not thread-safe; explore what would be needed to support concurrent readers
- [file] `hash-index-storage/test_bitcask.py` — Tests that exercise the read path, including edge cases like reading after compaction or reading tombstones

## Beliefs

- `bitcask-read-single-io` — `_read_record` performs exactly one `seek` and one `read` call per record retrieval, making point reads O(1) in I/O operations
- `bitcask-no-checksum-validation` — Records are read without any integrity check (no CRC, no checksum); corruption is detected only by the `assert` in `get()` comparing the expected vs actual key
- `bitcask-record-format-symmetry` — `_read_record` and `_write_record` share the same binary layout (`<dII` header + key bytes + value bytes); a change to either must be mirrored in the other
- `bitcask-tombstone-empty-string` — Deleted keys are represented as records with an empty-string value; `_read_record` returns `""` and the caller must interpret this as a deletion
- `bitcask-reader-not-threadsafe` — File handles in `self.file_handles` are shared and mutable (via `seek`), so concurrent calls to `_read_record` on the same `file_id` would race

