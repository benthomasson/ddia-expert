# Function: append in write-ahead-log/wal.py

**Date:** 2026-05-29
**Time:** 08:21

## `WriteAheadLog.append`

### Purpose

`append` writes a single operation record (PUT, DELETE, etc.) to the write-ahead log on disk. It exists to ensure durability before a mutation is applied to the main data structure — if the process crashes after `append` returns, the operation can be recovered from the log on restart.

### Contract

**Preconditions:**
- `op_type` must be a string key in `OP_BYTES` (one of `"PUT"`, `"DELETE"`, `"COMMIT"`, `"CHECKPOINT"`). No validation is performed — a missing key raises `KeyError` from the `OP_BYTES` dict lookup.
- `key` and `value` must be valid UTF-8 strings. The method calls `.encode("utf-8")` directly with no error handling.
- The WAL must be open (i.e., `self._fd` is not `None`). Calling after `close()` will raise `AttributeError`.

**Postconditions:**
- The record is written to the current WAL file.
- If `sync_mode == "sync"`, the data is `fsync`'d to disk before returning — a crash after return will not lose this record.
- If `sync_mode == "batch"`, the data *may* only be in the OS page cache. It is fsynced every `batch_sync_count` writes.
- The returned sequence number is globally unique and monotonically increasing within this WAL instance.

**Invariants:**
- `self._seq_num` increases by exactly 1 per `append` call.
- The lock ensures sequence number assignment and file write are atomic with respect to concurrent callers.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `op_type` | `str` | Operation name — `"PUT"`, `"DELETE"`, `"COMMIT"`, or `"CHECKPOINT"`. Looked up in `OP_BYTES` to get the integer tag for binary encoding. In practice callers use `"PUT"` or `"DELETE"` here; `"COMMIT"` and `"CHECKPOINT"` have dedicated methods. |
| `key` | `str` | The key being operated on. Encoded to UTF-8 bytes for the binary record. Cannot be `None`. |
| `value` | `str` | The value for PUT operations. Defaults to `""` for DELETE where no value is meaningful. Encoded to UTF-8. |

### Return Value

Returns `int` — the sequence number assigned to this record. Callers use this for:
- Tracking replay position (`replay(after_seq=N)` skips everything ≤ N)
- Truncation (`truncate(up_to_seq=N)` removes everything ≤ N)
- Ordering guarantees across concurrent writers

### Algorithm

1. **Acquire lock** — serializes all concurrent appends so sequence numbers are gap-free.
2. **Increment and capture sequence number** — `self._seq_num += 1` then snapshot to local `seq`. This two-step pattern avoids the value changing if another method were called (it can't under the lock, but the pattern is defensive).
3. **Encode the record** — `_encode_record` produces a binary blob: a 4-byte length prefix, CRC32 checksum, 8-byte sequence number, 1-byte op type, length-prefixed key, and length-prefixed value.
4. **Write to file** — a single `self._fd.write(data)` call. Because the file is opened in `"ab"` (append-binary) mode, the OS guarantees the write position is always at end-of-file, even across concurrent processes (though this implementation also serializes via the lock).
5. **Sync to disk** — `_do_sync()` either fsyncs immediately (`sync` mode) or batches fsyncs (`batch` mode). In `nosync` mode (any other string), nothing happens — data sits in the kernel buffer.
6. **Maybe rotate** — if the current file has grown past `_max_file_size` (default 10 MB), close it and open a new numbered WAL file. This keeps individual files manageable for replay and truncation.
7. **Return sequence number** under the lock — the caller knows this sequence number is durable (in sync mode).

### Side Effects

- **Disk I/O**: Writes bytes to the WAL file. In sync mode, also calls `fsync` which blocks until the disk controller acknowledges the write.
- **State mutation**: Increments `self._seq_num` and `self._write_count`. May replace `self._fd` and `self._current_file` if rotation triggers.
- **File creation**: Rotation creates a new `.wal` file in `self._dir`.

### Error Handling

The method does **no** error handling. Failures propagate directly:

- `KeyError` if `op_type` is not in `OP_BYTES`
- `UnicodeEncodeError` if `key`/`value` contain invalid surrogate characters
- `OSError` / `IOError` if the write or fsync fails (disk full, fd closed, etc.)
- `AttributeError` if called after `close()` (since `self._fd` is `None`)

Critically, if the write succeeds but `_do_sync()` raises, the sequence number has already been incremented but the caller receives an exception. The record may or may not be recoverable depending on whether the OS flushed the buffer before the crash.

### Usage Patterns

```python
wal = WriteAheadLog("/tmp/wal", sync_mode="sync")

# Typical put
seq = wal.append("PUT", "user:42", '{"name": "Alice"}')

# Delete (no value needed)
seq = wal.append("DELETE", "user:42")

# After applying to main store, truncate old entries
wal.truncate(up_to_seq=seq)
```

Individual `append` calls are independent — they don't participate in transactions. For atomic multi-operation writes, use `append_batch` instead, which writes all records plus a COMMIT marker under a single fsync.

### Dependencies

- `struct` / `zlib` — binary encoding and CRC32 checksums via `_encode_record`
- `os.fsync` — durability guarantee
- `threading.Lock` — concurrency control
- `OP_BYTES` — module-level reverse mapping from operation name strings to integer tags

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:append_batch` — Compare with `append`: how atomic multi-operation writes differ (buffered write, forced sync, COMMIT sentinel)
- [function] `write-ahead-log/wal.py:replay` — How recovery uses sequence numbers and handles uncommitted batches during crash recovery
- [function] `write-ahead-log/wal.py:_encode_record` — The binary record format: length prefix, CRC placement, and why the checksum covers op+key+value but not the sequence number
- [file] `write-ahead-log/test_wal.py` — Crash recovery and corruption test cases that validate the durability guarantees
- [general] `fsync-durability-guarantees` — The gap between `write()` and `fsync()` — what "batch" sync mode trades off and when data loss is possible

---

## Beliefs

- `wal-append-sequence-gap-free` — `append` increments the global sequence number by exactly 1 per call; sequence numbers are contiguous with no gaps (excluding batch operations which consume multiple).
- `wal-append-not-transactional` — Individual `append` calls are not wrapped in any transaction boundary; `replay()` returns all PUT/DELETE records regardless of whether they were part of a committed batch.
- `wal-sync-mode-durability` — In `"sync"` mode, `append` guarantees the record is fsync'd before returning; in `"batch"` mode, up to `batch_sync_count - 1` records may be lost on crash.
- `wal-append-no-validation` — `append` performs no validation of `op_type` against `OP_BYTES`; invalid operation names raise `KeyError` from the dictionary lookup, not a descriptive error.
- `wal-rotation-on-size` — After every `append`, the WAL checks whether the current file exceeds `max_file_size` and transparently rotates to a new numbered file if so.

