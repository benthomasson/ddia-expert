# Function: get in log-structured-hash-table/bitcask.py

**Date:** 2026-05-29
**Time:** 08:37

## `BitcaskStore.get` — Point lookup by key

### Purpose

`get` is the read path of the Bitcask storage engine. It retrieves a value for a given key by consulting the in-memory hash index to find the record's physical location (segment file + byte offset), then performing a single disk seek to read and validate the record. This is the core design principle of Bitcask: **O(1) index lookup in memory, one disk read per query**.

### Contract

- **Precondition**: `key` must be a string. The store must not be closed (no guard enforced — see assumptions below).
- **Postcondition**: Returns the raw value bytes if the key exists and the on-disk record is intact. Returns `None` if the key was never written or was deleted. Raises `CorruptionError` if the record on disk is truncated or fails its CRC check.
- **Invariant**: The method is read-only with respect to the index and segment files — it never mutates `self._index` or writes to disk.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `str` | The lookup key. Must match exactly (no prefix/range support). No length validation — any string the caller passes will be checked against the index. |

### Return Value

- `Optional[bytes]` — the raw value bytes on success, `None` on miss.
- The caller must handle `None` (key absent) and `CorruptionError` (data damaged). There is no distinction between "never written" and "deleted" — both return `None`, because `delete` removes the key from the index.

### Algorithm

1. **Index lookup** — Check `self._index` for `key`. If absent, short-circuit with `None`. This is a dict lookup: O(1) average case.

2. **Locate on disk** — Unpack `(seg_path, offset)` from the index entry. `seg_path` is the absolute path to the segment file; `offset` is the byte position where this record's header begins.

3. **Open a fresh file handle** — Opens `seg_path` in binary-read mode via a `with` block. This is deliberate: the class also maintains a `_file_handles` cache elsewhere, but `get` avoids it. The comment says "to avoid position conflicts" — because a cached handle's file position would be shared across calls, causing reads to land at wrong offsets under interleaved access.

4. **Read the header** — Seeks to `offset`, reads `HEADER_SIZE` (12 bytes: three unsigned 32-bit ints in network byte order). Unpacks into `(crc, key_size, value_size)`.

5. **Read the payload** — Reads `key_size + value_size` bytes immediately following the header. The payload is `key_bytes || value_bytes` concatenated.

6. **CRC validation** — Computes `zlib.crc32` over the payload and masks to 32 bits. Compares against the stored `crc`. This detects bit-rot, partial writes, and file corruption.

7. **Extract value** — Returns `payload[key_size:]`, slicing off the key prefix to yield only the value bytes.

### Side Effects

Minimal. Opens and closes a file descriptor on each call (the `with` block). No mutations to `self._index`, no writes to disk, no network I/O. The file handle is not cached — each `get` pays the open/close cost but avoids stale-position bugs.

### Error Handling

| Condition | Behavior |
|-----------|----------|
| Key not in index | Returns `None` (normal miss) |
| Header read returns fewer than 12 bytes | Raises `CorruptionError` |
| Payload read returns fewer than expected bytes | Raises `CorruptionError` |
| CRC mismatch | Raises `CorruptionError` with expected vs actual CRC |
| Segment file missing (deleted or moved) | Unhandled `FileNotFoundError` propagates to caller |
| Key was deleted | Returns `None` — `delete()` removes the index entry, so it never reaches disk |

Note that `get` does **not** check whether the value is a tombstone (`TOMBSTONE`). It doesn't need to: `delete()` removes the key from the index, so a tombstone record is never reachable through `get`. The tombstone sentinel only matters during recovery (`_scan_segment`) and compaction.

### Usage Patterns

```python
store = BitcaskStore("/tmp/mydb")
store.put("user:1", b'{"name": "Alice"}')
value = store.get("user:1")    # b'{"name": "Alice"}'
value = store.get("missing")   # None
```

Callers should:
- Handle `None` for missing keys
- Catch `CorruptionError` if they want graceful degradation instead of a crash
- Not assume thread safety — multiple threads calling `get` concurrently is safe (fresh handles), but concurrent `get` + `put` with the same key could read stale data if the index update and file write interleave

### Dependencies

- `struct` — binary (de)serialization of the fixed-size header
- `zlib.crc32` — checksum computation for integrity verification
- `self._index` — the in-memory hash map, populated by `_recover()` at startup and maintained by `put()`/`delete()`
- On-disk segment files — the actual data store; `get` trusts that the file at `seg_path` still contains the record written at `offset`

### Assumptions not enforced by the type system

1. **The segment file still exists at `seg_path`** — compaction deletes old segments and updates the index, but if the process crashes between deletion and index update, `get` will raise `FileNotFoundError`.
2. **The store hasn't been `close()`d** — no guard prevents calling `get` after `close()`.
3. **`offset` is a valid position** — the index is trusted implicitly; a corrupted index would cause reads at arbitrary offsets.
4. **Keys are valid UTF-8** — `get` never decodes the key (it only slices past it), but the write path encodes with UTF-8, so the stored key length in bytes may differ from `len(key)` in characters.
5. **Values are never equal to `TOMBSTONE`** — if a caller stores the literal bytes `b"__BITCASK_TOMBSTONE__"` as a value, recovery will treat it as a deletion.

---

## Topics to Explore

- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — How the index is rebuilt from raw segment data on startup; the read-path mirror of `get` that must also handle tombstones
- [function] `log-structured-hash-table/bitcask.py:compact` — How frozen segments are merged and old entries discarded, and how the index is atomically updated to point to new offsets
- [function] `log-structured-hash-table/bitcask.py:_write_record` — The write-path counterpart that produces the exact binary layout `get` expects to parse
- [file] `log-structured-hash-table/test_bitcask.py` — Test coverage showing the put/get/delete/compact lifecycle and corruption edge cases
- [general] `bitcask-crash-recovery` — What happens if the process crashes mid-write: how partial records at the end of a segment are handled during recovery vs during `get`

## Beliefs

- `bitcask-get-single-seek` — `get` performs exactly one disk seek per call; the key-to-offset mapping is resolved entirely in memory via `self._index`
- `bitcask-get-fresh-handle` — `get` opens a new file handle on every call rather than reusing cached handles, trading file-descriptor overhead for correctness under concurrent access
- `bitcask-tombstone-invisible-to-get` — `get` never encounters tombstone records because `delete()` removes the key from the index; tombstone handling is solely a recovery/compaction concern
- `bitcask-crc-covers-payload-not-header` — The CRC32 checksum covers only the key+value payload, not the header itself; a corrupted `key_size` or `value_size` field would cause a wrong-length read rather than a CRC failure
- `bitcask-get-no-file-existence-guard` — If a segment file is deleted out-of-band (or by a crash during compaction), `get` will raise an unhandled `FileNotFoundError` rather than returning `None`

