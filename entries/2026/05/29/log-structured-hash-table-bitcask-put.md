# Function: put in log-structured-hash-table/bitcask.py

**Date:** 2026-05-29
**Time:** 06:45

## `BitcaskStore.put`

### Purpose

`put` is the primary write path for this Bitcask-style key-value store. It appends a key-value record to the active segment file and updates the in-memory hash index so subsequent reads can locate the value by seeking directly to its byte offset. This is the core operation that makes Bitcask an *append-only, log-structured* store — writes never modify existing data on disk, they only append.

### Contract

**Preconditions:**
- The store must be initialized (`_active_file` is open for appending, `_active_path` is set).
- `key` must be a valid UTF-8 string (it gets `.encode("utf-8")` inside `_write_record`).
- `value` must not be `TOMBSTONE` (`b"__BITCASK_TOMBSTONE__"`) — that sentinel is reserved for `delete()`. This is **not enforced** by `put`.

**Postconditions:**
- The record is durably written to the active segment (flushed to OS buffers via `flush()`).
- `self._index[key]` points to `(active_segment_path, byte_offset)` of the new record.
- If the active segment exceeded `max_segment_size`, a new segment was opened *before* the write.
- If the number of frozen segments hit `auto_compact_threshold`, compaction ran.

**Invariants maintained:**
- The in-memory index always maps each live key to its *most recent* record location.
- The active segment is always the highest-numbered segment file.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `str` | The lookup key. Encoded to UTF-8 for on-disk storage. No length limit is enforced, but very large keys inflate every record's header payload. |
| `value` | `bytes` | The raw value payload. Arbitrary bytes. Callers must not pass `TOMBSTONE` — doing so would corrupt the logical state (the index would map the key to a tombstone record that `get` would return as data). |

### Return Value

Returns `int` — the byte offset within the active segment where the record header begins. This is the same offset stored in the index. Callers don't typically need this; it's useful for diagnostics or for building external secondary indexes.

### Algorithm

1. **Rotation check** — Query the active file's current write position (`tell()`). If it's at or past `max_segment_size`, close the current segment and open a new one via `_rotate_segment()`. This ensures no single segment grows unboundedly.

2. **Write the record** — Call `_write_record(key, value)`, which:
   - Encodes the key to UTF-8, concatenates with value to form the payload.
   - Computes a CRC32 checksum over the payload.
   - Packs a fixed-size header (`!III`: crc, key_len, value_len).
   - Appends header + payload to the active file and flushes.
   - Returns the byte offset where the header started.

3. **Invalidate stale read handle** — Removes any cached read file handle for the active segment path from `_file_handles`. After appending, a previously-opened read handle would have a stale file position. (Note: `get()` actually opens a fresh handle each time via `with open(...)`, so this is defensive cleanup — it prevents `_get_read_handle` from returning a handle whose cursor is in the wrong place.)

4. **Update the index** — Sets `self._index[key] = (active_path, offset)`. If the key already existed, this silently overwrites the old entry. The old record remains on disk as dead data until compaction reclaims it.

5. **Auto-compact check** — Counts frozen (non-active) segments. If the count meets or exceeds `auto_compact_threshold`, triggers `compact()` to merge all frozen segments, removing stale records and tombstones.

6. **Return the offset.**

### Side Effects

- **Disk I/O**: Appends bytes to the active segment file and flushes.
- **Segment rotation**: May create a new segment file on disk.
- **Index mutation**: Overwrites the previous index entry for this key (if any).
- **File handle eviction**: Closes and removes any cached read handle for the active segment.
- **Compaction cascade**: May trigger a full compaction cycle, which deletes old segment files, creates a compacted segment, renames the active segment, and rewrites large portions of the index. This means a single `put` call can have significant latency if it triggers compaction.

### Error Handling

No exceptions are explicitly caught. Possible failures:

- **`OSError`/`IOError`**: If the filesystem is full, permissions are wrong, or the file was closed externally. These propagate uncaught — a failed write leaves the store in an inconsistent state (the record may be partially written but the index may or may not be updated depending on where the exception occurred).
- **`UnicodeEncodeError`**: If `key` contains characters that can't be encoded to UTF-8 (unlikely with Python `str`, but possible with surrogates).
- There is **no write-ahead log or atomicity guarantee** — a crash between `_write_record` and the index update means the record is on disk but not indexed (it will be recovered on next startup via `_recover`).

### Usage Patterns

```python
store = BitcaskStore("/tmp/mydb", max_segment_size=4096)
store.put("user:1", b'{"name": "Alice"}')
store.put("user:2", b'{"name": "Bob"}')
store.put("user:1", b'{"name": "Alice Updated"}')  # overwrites previous
```

Callers must ensure the store is open (not `close()`'d). There is no locking — concurrent `put` calls from multiple threads will corrupt both the file and the index. The caller is responsible for external synchronization if needed.

### Dependencies

- **`struct`**: Binary packing for the record header format (`!III`).
- **`zlib.crc32`**: Integrity checksum over key+value payload.
- **`os`**: Filesystem operations (directory creation, file size, deletion during compaction).
- **Internal**: `_write_record`, `_rotate_segment`, `_frozen_segment_paths`, `compact` — the write path is distributed across these helpers.

---

## Topics to Explore

- [function] `log-structured-hash-table/bitcask.py:compact` — The compaction algorithm that reclaims dead space from frozen segments; understanding it is essential to see how `put`'s auto-compact trigger affects write latency
- [function] `log-structured-hash-table/bitcask.py:_recover` — Startup recovery that replays all segments to rebuild the index; this is what makes the append-only design crash-safe
- [function] `log-structured-hash-table/bitcask.py:get` — The read path that uses the index entries `put` creates; note that it opens a fresh file handle each time rather than using the cached handles
- [file] `log-structured-hash-table/test_bitcask.py` — Tests that exercise segment rotation, compaction triggering, and overwrite semantics
- [general] `bitcask-vs-lsm-tree` — How this hash-index approach (O(1) reads, all keys in memory) compares to the LSM-tree in `log-structured-merge-tree/`, which trades write amplification for range query support

---

## Beliefs

- `put-append-only` — `put` never modifies existing on-disk data; overwrites create a new record and update only the in-memory index, leaving the old record as reclaimable garbage
- `put-triggers-compaction` — A `put` call can trigger synchronous compaction (file deletion, rewriting, renaming) if frozen segment count reaches `auto_compact_threshold`, making write latency non-uniform
- `put-no-tombstone-guard` — `put` does not reject `TOMBSTONE` as a value; passing it creates a record indistinguishable from a delete, which would silently corrupt the key's state on recovery
- `put-not-thread-safe` — There is no locking or synchronization in the write path; concurrent `put` calls will interleave writes to the active file and corrupt the index
- `put-crash-recovery-gap` — If a crash occurs after `_write_record` but before the index update, the record is orphaned on disk until `_recover` replays the segment on next startup

