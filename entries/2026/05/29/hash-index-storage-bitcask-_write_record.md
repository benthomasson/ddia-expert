# Function: _write_record in hash-index-storage/bitcask.py

**Date:** 2026-05-29
**Time:** 08:15

## `_write_record` — Append a key-value record to the active data file

### Purpose

`_write_record` is the single write path for all mutations in this Bitcask store. Every `put` and `delete` flows through it. It serializes a key-value pair into a binary record format (header + key + value), appends it to the currently active data file, and returns the metadata the caller needs to update the in-memory index (`keydir`).

This method only handles the physical write. It does **not** update the `keydir` — that's the caller's responsibility, which is a deliberate separation of concerns. The method also doesn't handle file rotation; `_maybe_rotate` is called by the caller before invoking this.

### Contract

**Preconditions:**
- `self.active_file` is open in append-binary mode (`"ab"`) and positioned at the end of the file.
- `key` and `value` are valid Python strings (UTF-8 encodable). There is no validation — non-encodable strings will raise.
- The caller has already called `_maybe_rotate()` if size limits matter.

**Postconditions:**
- Exactly one record (header + key bytes + value bytes) has been appended to the active file.
- The file's write position has advanced by `len(record)` bytes.
- The data is flushed to the OS buffer (`flush()`), and optionally fsynced to disk.
- The returned `offset` points to the start of the newly written record in the active file.

**Invariants:**
- Records are never overwritten — the file is append-only.
- The on-disk record format is: `[timestamp: float64][key_size: uint32][val_size: uint32][key_bytes][val_bytes]`.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `str` | The key to store. Encoded to UTF-8 for serialization. No length limit enforced — a key larger than `max_file_size` would silently succeed. |
| `value` | `str` | The value to store. An empty string `""` is the tombstone convention used by `delete()`. |

### Return Value

Returns `tuple[int, int, float]`:

| Index | Name | Meaning |
|-------|------|---------|
| 0 | `offset` | Byte position where this record starts in the active file. Used as the disk pointer in `keydir`. |
| 1 | `size` | Total byte length of the record (header + key + value). Used to read the record back in `_read_record`. |
| 2 | `ts` | The `time.time()` timestamp captured at the start of the write. Used for conflict resolution during compaction (latest timestamp wins). |

The caller must use these to construct a `KeyEntry` and update `keydir`, or (for deletes) to pop the key from `keydir`.

### Algorithm

1. **Capture timestamp** — `time.time()` gives wall-clock seconds as a float. This happens first, before any I/O, so the timestamp reflects intent time, not write-completion time.

2. **Encode key and value** — Both are UTF-8 encoded to raw bytes. The byte lengths are needed for the header.

3. **Pack the header** — Uses `struct.pack` with format `<dII` (little-endian: 8-byte double for timestamp, two 4-byte unsigned ints for key/value sizes). This produces exactly 16 bytes (`HEADER_SIZE`).

4. **Assemble the record** — Simple concatenation: `header + key_bytes + val_bytes`. No CRC or checksum — the implementation trusts the filesystem.

5. **Capture current offset** — `self.active_file.tell()` gives the byte position where the record will land. Because the file is opened in append mode, this is always the end of the file.

6. **Write** — The entire record is written in one `write()` call, which on most OSes is atomic for reasonable sizes (below `PIPE_BUF`), but there's no explicit guarantee here for very large records.

7. **Flush** — `flush()` pushes Python's userspace buffer to the OS. This is always done.

8. **Optional fsync** — If `self.sync_writes` is `True` (the default), `os.fsync()` forces the OS to write through to the physical disk. This is the durability guarantee — without it, a crash could lose recently written records that were still in the OS page cache.

9. **Return metadata** — The offset, total record size, and timestamp are returned for the caller to index.

### Side Effects

- **Disk I/O**: Appends bytes to the active `.data` file. This is the primary mutation.
- **fsync** (conditional): Forces a disk sync, which is expensive — this is the main performance knob in Bitcask. Setting `sync_writes=False` trades durability for throughput.
- **File position**: The active file's write cursor advances. Future calls to `tell()` will return a higher offset.
- **No index mutation**: `keydir` is untouched — this is purely a log-append operation.

### Error Handling

There is no explicit error handling. The following can propagate to the caller:

- `UnicodeEncodeError` — if `key` or `value` contains characters that can't be UTF-8 encoded (unlikely for Python `str`, but possible with surrogates).
- `OSError` / `IOError` — if the write, flush, or fsync fails (disk full, file closed, I/O error).
- `struct.error` — if key or value byte lengths exceed `2^32 - 1` (the uint32 max), the `struct.pack` will raise.

None of these are caught — they bubble up through `put()` or `delete()` to the application. A partial write (crash mid-write) would leave a truncated record at the end of the file, which `_scan_data_file` would hit as a short read and silently stop scanning (the `if len(header_data) < HEADER_SIZE: break` guard).

### Usage Patterns

Called in exactly two places:

- **`put(key, value)`** — writes the record, then updates `keydir` with a new `KeyEntry` pointing to it.
- **`delete(key)`** — writes a tombstone (empty value `""`), then removes the key from `keydir`.

The caller always calls `_maybe_rotate()` first to ensure the active file hasn't exceeded `max_file_size`. This ordering matters — if rotation happens after the write, the record lands in an oversized file but still works.

### Dependencies

| Dependency | Usage |
|------------|-------|
| `time.time()` | Monotonically-ish increasing wall clock for timestamps. Not monotonic — clock adjustments can produce out-of-order timestamps, which would confuse compaction's "latest wins" logic. |
| `struct` | Binary packing with `HEADER_FORMAT = "<dII"` (16 bytes). |
| `os.fsync` | Durability guarantee when `sync_writes` is enabled. |
| `self.active_file` | Must be an open file handle in `"ab"` mode. Managed by `_open_active_file()`. |

### Notable Design Choices

- **No CRC/checksum**: Real Bitcask implementations include a CRC32 in the header for corruption detection. This implementation trusts the filesystem, which simplifies the code but means silent corruption goes undetected.
- **No write-ahead log or double-write**: A crash during `write()` can leave a partial record. Recovery relies on `_scan_data_file` stopping at the truncated tail.
- **Timestamp from `time.time()`**: Wall-clock timestamps are used for "latest wins" during compaction, but `time.time()` is not monotonic — NTP adjustments or manual clock changes could cause a newer write to have an older timestamp.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:_read_record` — The inverse operation: how it uses the offset/size to deserialize and verify the key
- [function] `hash-index-storage/bitcask.py:compact` — How stale records are garbage-collected and why the timestamp in `_write_record` matters for "latest wins" resolution
- [function] `hash-index-storage/bitcask.py:_scan_data_file` — How the index is rebuilt by replaying the append-only log, and how it handles tombstones and truncated tails
- [general] `bitcask-durability-tradeoffs` — The relationship between `sync_writes`, `flush()`, and `fsync()` — and what guarantees each level actually provides on different filesystems
- [general] `bitcask-paper-vs-implementation` — How this implementation compares to the original Bitcask paper (missing CRC, merge triggers, expiry)

---

## Beliefs

- `write-record-is-append-only` — `_write_record` only appends to the active file; it never overwrites or seeks backward, preserving the append-only log invariant.
- `write-record-no-index-mutation` — `_write_record` does not modify `keydir`; the caller is responsible for updating the in-memory index after the write returns.
- `tombstone-convention-empty-value` — Deletions are encoded as records with an empty value string (`""`); there is no separate tombstone marker byte or flag.
- `no-crc-on-records` — Records have no checksum or CRC field, so on-disk corruption cannot be detected during reads or compaction.
- `fsync-controlled-by-sync-writes` — Durability depends on the `sync_writes` flag: when `True`, every write is fsynced to disk; when `False`, data may be lost on crash.

