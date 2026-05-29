# Function: append_batch in write-ahead-log/wal.py

**Date:** 2026-05-28
**Time:** 18:45

## `append_batch` — Atomic Batch Write with Commit Marker

### Purpose

`append_batch` writes a group of operations (PUTs and DELETEs) to the write-ahead log as a single atomic unit, terminated by a COMMIT record. This is the transactional write path — either all operations in the batch are durable and committed, or none of them are (if a crash occurs before the fsync completes). It exists to support multi-key transactions where partial application would leave the database in an inconsistent state.

### Contract

**Preconditions:**
- The WAL must be open (`self._fd` is not `None`). No guard exists — calling after `close()` will raise `AttributeError`.
- Each tuple in `operations` must be `(op_type, key, value)` where `op_type` is a string key in `OP_BYTES` (i.e., `"PUT"`, `"DELETE"`, `"COMMIT"`, or `"CHECKPOINT"`). Passing an invalid op_type raises `KeyError`.
- Keys and values must be valid UTF-8 strings (they're encoded with `str.encode("utf-8")`).

**Postconditions:**
- All operation records plus a trailing COMMIT record have been written and fsynced to disk.
- `self._seq_num` has advanced by `len(operations) + 1` (one per operation, one for the COMMIT).
- The file may have been rotated if it exceeded `_max_file_size`.

**Invariants:**
- The lock guarantees no interleaving with other `append`/`append_batch`/`checkpoint` calls — the batch is contiguous on disk.
- Sequence numbers are strictly monotonically increasing across all records in the WAL.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `operations` | `List[Tuple[str, str, str]]` | List of `(op_type, key, value)` tuples. An empty list is technically valid — it would write a lone COMMIT record. The code does not guard against this. |

### Return Value

Returns `int` — the sequence number assigned to the COMMIT record. This is the highest sequence number in the batch and serves as the batch's identity. Callers can pass this to `truncate(up_to_seq)` to garbage-collect this batch after it has been applied to the main data store.

### Algorithm

1. **Acquire the lock** — prevents concurrent writes from interleaving records.
2. **Build a buffer** — allocates a `bytearray` to accumulate all encoded records before any I/O.
3. **Encode each operation** — for each `(op_type, key, value)`:
   - Increment `_seq_num` and assign it to this record.
   - Encode via `_encode_record` (length-prefixed binary with CRC32 checksum) and append to buffer.
4. **Append a COMMIT sentinel** — increment `_seq_num` once more, encode a COMMIT record with empty key/value, append to buffer.
5. **Single write** — `self._fd.write(bytes(buf))` sends the entire batch to the OS in one `write()` call. This doesn't guarantee atomicity at the filesystem level, but it minimizes the window for partial writes.
6. **Force fsync** — `_do_sync(force=True)` unconditionally flushes and fsyncs, regardless of the configured sync mode. Batches always force durability.
7. **Maybe rotate** — if the file has grown past `_max_file_size`, close it and open a new numbered WAL file.
8. **Return the COMMIT sequence number**.

### Side Effects

- **Disk I/O**: Writes binary data and calls `fsync`. This is the expensive part — `fsync` blocks until the OS confirms data is on stable storage.
- **Sequence number mutation**: `self._seq_num` advances by `len(operations) + 1`. This is not rolled back on failure.
- **File rotation**: May close the current file handle and open a new one, changing `self._current_file` and `self._fd`.
- **Thread lock held**: The lock is held across the entire I/O path including fsync, which means other writers block for the full duration of the disk sync.

### Error Handling

The method has **no try/except blocks**. Failures propagate directly to the caller:

- `KeyError` if `op_type` is not in `OP_BYTES`.
- `OSError` / `IOError` from `write()` or `fsync()` if the disk is full or the file descriptor is invalid.
- `AttributeError` if `self._fd` is `None` (WAL already closed).
- `UnicodeEncodeError` if key/value contain surrogates or other non-encodable content (unlikely for normal strings).

**Critical concern**: if `write()` succeeds but `fsync()` fails, the sequence numbers have already been incremented and the partial data may be in the OS page cache. There's no rollback. On recovery, `_read_record` would see partial/corrupt data and `_recover_seq_num` would scan past it (the `while True` loop in `_recover_seq_num` breaks on `ValueError` from CRC mismatch), but the gap in sequence numbers is permanent.

### Usage Patterns

```python
wal = WriteAheadLog("/tmp/wal_dir")

# Typical transactional write
commit_seq = wal.append_batch([
    ("PUT", "account:alice", "balance:900"),
    ("PUT", "account:bob", "balance:1100"),
])

# After applying to the main store, truncate
wal.truncate(commit_seq)
```

Callers are responsible for:
1. Not calling after `close()`.
2. Using the returned sequence number to track what has been applied.
3. Handling I/O exceptions if disk failures are possible.

### Dependencies

| Dependency | Usage |
|------------|-------|
| `_encode_record` (module-level) | Serializes each record to the binary wire format with CRC32 |
| `OP_BYTES` (module-level dict) | Maps string op names → integer op codes |
| `OP_COMMIT` (module-level constant, value `3`) | The commit sentinel op code |
| `threading.Lock` | Mutual exclusion across threads |
| `os.fsync` | Durability guarantee |
| `_do_sync`, `_maybe_rotate` | Internal helpers for sync policy and file management |

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:replay` — How committed vs uncommitted batches are distinguished during recovery (notably, the current `replay` implementation does *not* enforce commit boundaries — it returns all PUT/DELETE records regardless of whether a COMMIT follows them)
- [function] `write-ahead-log/wal.py:_encode_record` — The binary format and CRC scheme that each record in the batch relies on
- [function] `write-ahead-log/wal.py:truncate` — How applied batches are garbage-collected by rewriting WAL files in place
- [general] `wal-atomicity-guarantees` — Whether a single `write()` call actually provides atomicity on common filesystems (ext4, APFS) and what happens with partial writes across page boundaries
- [file] `event-sourcing-store/test_event_store.py` — How the event sourcing layer uses `append_batch` to persist multi-event transactions

---

## Beliefs

- `wal-batch-commit-sentinel` — Every `append_batch` call writes exactly one COMMIT record as the final record in the batch, with empty key and value fields
- `wal-batch-forces-fsync` — `append_batch` always calls `_do_sync(force=True)`, bypassing the batch sync counter, so every batch is immediately durable regardless of the WAL's configured sync mode
- `wal-batch-single-write` — All records in a batch (operations + COMMIT) are buffered into a single `bytearray` and written in one `write()` call to minimize partial-write risk
- `wal-replay-ignores-commit-boundaries` — The `replay` method returns all PUT/DELETE records after `after_seq` without verifying that a COMMIT record follows them, meaning uncommitted batches are replayed identically to committed ones
- `wal-no-rollback-on-io-failure` — If `write()` or `fsync()` fails mid-batch, sequence numbers are already incremented and no rollback occurs, creating a permanent gap in the sequence space

