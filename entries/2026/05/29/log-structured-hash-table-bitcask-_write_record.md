# Function: _write_record in log-structured-hash-table/bitcask.py

**Date:** 2026-05-29
**Time:** 07:27

## `_write_record` — Append a single key-value record to the active segment

### Purpose

`_write_record` is the low-level write primitive for the Bitcask store. Every mutation — `put`, `delete`, and compaction — ultimately serializes data to disk through this method. It encodes a key-value pair into the on-disk binary format (header + payload), appends it to the current active segment file, and returns the byte offset where the record begins. That offset is what the in-memory hash index stores to locate the record later.

### Contract

**Preconditions:**
- `self._active_file` is open in append-binary mode (`"ab"`) and is not `None`.
- `key` is a valid UTF-8 string. No length validation is performed — arbitrarily large keys are accepted.
- `value` is raw bytes. Can be arbitrary content, including the sentinel `TOMBSTONE` value (callers decide the semantics).

**Postconditions:**
- Exactly `HEADER_SIZE + len(key_bytes) + len(value)` bytes have been appended to the active segment file.
- The file buffer has been flushed to the OS (though not necessarily `fsync`'d to disk).
- The returned offset points to the first byte of the header for this record.

**Invariant:** The on-disk record is self-describing — the header contains enough information (key size, value size) to read the record without external metadata, and the CRC covers the full payload for integrity verification on read.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `str` | The logical key. Encoded to UTF-8 bytes internally. No maximum length enforced. |
| `value` | `bytes` | The raw value to store. May be actual user data or the `TOMBSTONE` sentinel. |

**Edge cases:** An empty string key (`""`) produces zero key bytes but is technically valid. An empty `value` (`b""`) is also valid — the CRC and sizes will reflect zero-length value.

### Return Value

Returns `int` — the byte offset within the active segment file where this record's header starts. The caller uses this offset (paired with the file path) to build the index entry for O(1) lookups. The caller is responsible for updating `self._index`; this method does not touch the index.

### Algorithm

```
1. Encode the key string to UTF-8 bytes.
2. Concatenate key_bytes + value to form the payload.
3. Compute CRC-32 over the payload, masked to 32 bits unsigned.
4. Pack a 12-byte header: [crc32 | key_size | value_size] in network byte order (!III).
5. Capture the current file position (this is the record's offset).
6. Write header + payload as a single contiguous write.
7. Flush the write buffer.
8. Return the captured offset.
```

The CRC covers only the payload (key bytes + value), not the header itself. This means a corrupted header (e.g., wrong sizes) won't be caught by the CRC — but it will cause the reader to extract the wrong payload slice, which will then fail the CRC check indirectly.

### Side Effects

- **Disk I/O:** Appends bytes to `self._active_file` and flushes. This is the only method that writes data records to disk.
- **File position advancement:** After the call, `self._active_file.tell()` has advanced by the record size. This is how `put` detects when the segment exceeds `_max_segment_size` and needs rotation.
- **No index mutation:** Deliberately does not update `self._index`. The caller (`put`, `delete`, `compact`) decides whether and how to update the index.
- **No fsync:** `flush()` pushes data from Python's buffer to the OS kernel buffer, but a crash before the OS writes to disk could lose the record. This is a durability trade-off for write throughput.

### Error Handling

This method does not catch any exceptions. Possible failures include:

- `UnicodeEncodeError` if `key` contains characters that can't be encoded as UTF-8 (shouldn't happen for normal Python `str`).
- `OSError`/`IOError` if the underlying file write or flush fails (disk full, file closed, permission error).
- `struct.error` if the sizes overflow the `!III` format (each field is an unsigned 32-bit int, so keys or values larger than ~4 GB would fail).

All of these propagate to the caller unhandled.

### Usage Patterns

Called from three sites:

1. **`put(key, value)`** — writes user data, then updates the index with the returned offset.
2. **`delete(key)`** — writes the key with `TOMBSTONE` as value, then removes the key from the index.
3. **`compact()`** — rewrites live records to a new segment (though compaction actually bypasses `_write_record` and writes directly — it re-serializes records inline in the compaction loop).

The leading underscore signals this is an internal method. Callers must handle segment rotation *before* calling `_write_record` — the method itself has no size-checking logic.

### Dependencies

- **`struct`** — binary packing with format `"!III"` (network-order, three unsigned 32-bit ints = 12 bytes).
- **`zlib.crc32`** — CRC-32 checksum for integrity. The `& 0xFFFFFFFF` mask ensures a positive unsigned value on all Python versions (Python 2's `crc32` could return signed values; this is defensive).
- **`self._active_file`** — an open file handle in `"ab"` mode, managed by `_open_new_segment` and `_rotate_segment`.

### On-Disk Record Layout

```
┌──────────────────────── HEADER (12 bytes) ─────────────────────────┐
│  CRC32 (4B)  │  key_size (4B)  │  value_size (4B)                 │
├──────────────────────── PAYLOAD (variable) ────────────────────────┤
│  key_bytes (key_size B)  │  value_bytes (value_size B)            │
└───────────────────────────────────────────────────────────────────-┘
```

This is the exact format that `_scan_segment` and `get` expect when reading back.

---

## Topics to Explore

- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — The read-side mirror of `_write_record`; reconstructs the index by parsing the same binary format
- [function] `log-structured-hash-table/bitcask.py:compact` — Rewrites live records to eliminate stale data; notable because it re-serializes records directly rather than calling `_write_record`
- [function] `log-structured-hash-table/bitcask.py:get` — Shows how the offset returned by `_write_record` is used for point lookups with CRC verification
- [general] `bitcask-durability-guarantees` — The gap between `flush()` and `fsync()` and what it means for crash recovery in this implementation
- [general] `bitcask-paper-comparison` — How this implementation compares to the original Riak/Bitcask design, particularly around hint files and merge behavior

## Beliefs

- `write-record-no-index-update` — `_write_record` only appends to disk and returns an offset; it never modifies `self._index`, leaving index management to callers
- `crc-covers-payload-not-header` — The CRC-32 is computed over `key_bytes + value` only; header corruption is detected indirectly when the wrong payload slice fails the CRC check on read
- `flush-not-fsync` — `_write_record` calls `flush()` but not `os.fsync()`, so records can be lost in an OS crash even after a successful write returns
- `record-format-self-describing` — Each on-disk record contains key_size and value_size in the header, so it can be read without any external metadata or index
- `compact-bypasses-write-record` — The `compact` method re-serializes records directly to the output file rather than calling `_write_record`, duplicating the serialization logic

