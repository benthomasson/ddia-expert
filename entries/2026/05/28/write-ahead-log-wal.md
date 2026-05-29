# File: write-ahead-log/wal.py

**Date:** 2026-05-28
**Time:** 18:07

# `write-ahead-log/wal.py` — Write-Ahead Log for Crash Recovery

## Purpose

This file implements a **Write-Ahead Log (WAL)**, the foundational durability primitive described in DDIA Chapter 3. A WAL ensures that every mutation is written to a sequential, append-only log on disk *before* it's applied to the main data structure. If the process crashes, the log can be replayed to reconstruct state that was acknowledged but not yet persisted.

This implementation owns: sequential record encoding/decoding, fsync-based durability guarantees, log file rotation, truncation after checkpointing, and crash-safe replay.

## Key Components

### Constants & Mappings

`OP_PUT`, `OP_DELETE`, `OP_COMMIT`, `OP_CHECKPOINT` are the four operation types, stored as single-byte integers in the binary format. `OP_NAMES` and `OP_BYTES` provide bidirectional lookup between the integer wire format and string names used in the public API.

### `WALRecord` (dataclass)

The deserialized form of a single log entry. Carries `seq_num` (monotonic ordering), `op_type` (string name), `key`, `value`, and `checksum`. This is the unit of data returned by `replay()` and `iterate()`.

### `_encode_record(seq_num, op_type_byte, key, value) -> bytes`

Serializes a record into a length-prefixed binary frame:

```
[4B record_length][4B CRC32][8B seq_num][1B op_type][4B key_len][key][4B val_len][value]
```

The total frame on disk is `4 (length prefix) + record_length` bytes. The CRC covers `op_type + key + value` — notably it does **not** cover `seq_num` or lengths, so a corrupted sequence number would not be detected by the checksum.

### `_read_record(f) -> Optional[WALRecord]`

Reads one record from a file handle. Returns `None` on EOF or partial read (torn write), raises `ValueError` on CRC mismatch. This distinction is critical: a short read means "the process crashed mid-write, ignore this tail" while a CRC failure means "data corruption in the middle of the log."

### `WriteAheadLog` (class)

The main class. Constructor parameters:

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `log_dir` | required | Directory for `.wal` files |
| `sync_mode` | `"sync"` | `"sync"` = fsync every write; `"batch"` = fsync every N writes; `"none"` = never fsync |
| `max_file_size` | 10 MB | Rotation threshold |
| `batch_sync_count` | 100 | Writes between fsyncs in batch mode |

**Core methods:**

- **`append(op_type, key, value) -> int`** — Write a single record. Returns the assigned sequence number. Fsyncs per the sync mode.
- **`append_batch(operations) -> int`** — Atomically write multiple operations terminated by a `COMMIT` record. Always force-fsyncs regardless of sync mode. Returns the COMMIT's sequence number.
- **`checkpoint() -> int`** — Write a `CHECKPOINT` marker. This signals that all prior records have been applied to the main store and can be safely truncated.
- **`truncate(up_to_seq)`** — Remove all records with `seq_num <= up_to_seq`. Files that become empty are deleted; files with remaining records are rewritten in place.
- **`replay(after_seq) -> List[WALRecord]`** — Return all `PUT`/`DELETE` records with `seq_num > after_seq`. Filters out `COMMIT` and `CHECKPOINT` records.
- **`iterate() -> Iterator[WALRecord]`** — Yield all raw records including `COMMIT`/`CHECKPOINT`, without filtering.

## Patterns

**Length-prefixed framing.** Each record starts with a 4-byte length, so the reader can skip or validate entire records without parsing internals. This is standard in log-structured storage (Kafka, LevelDB WAL, etc.).

**CRC integrity checking.** Every record carries a CRC32 computed over the payload. This detects bit rot and torn writes where the length was written but the body is garbage.

**Monotonic sequence numbers.** `_seq_num` increments under the lock and is never reused. On recovery, `_recover_seq_num` scans all files to find the high-water mark, so new records continue the sequence without gaps or collisions.

**File rotation.** When a WAL file exceeds `max_file_size`, it's fsynced, closed, and a new file with an incremented numeric name is created. This bounds individual file sizes and enables efficient truncation (delete whole files rather than rewriting).

**Coarse-grained locking.** A single `threading.Lock` serializes all writes. This is simple and correct — WAL writes are inherently sequential on disk, so fine-grained locking would add complexity without improving throughput.

**Sync mode trade-off.** The three sync modes (`sync`, `batch`, `none`) let callers choose their position on the durability-vs-performance spectrum. `append_batch` always force-fsyncs because batch atomicity requires it — a crash between writing the operations and the COMMIT marker would leave an uncommitted batch.

## Dependencies

**Imports:** `os` (file I/O, fsync), `struct` (binary packing), `zlib` (CRC32), `threading` (lock), plus standard library typing/dataclass support.

**Imported by:** `test_wal.py` and `tester_test_wal.py` — this is a leaf module with no downstream production consumers in this repo. It exists as a standalone reference implementation.

## Flow

### Write path

1. Caller invokes `append()` or `append_batch()`
2. Lock acquired
3. Sequence number(s) incremented
4. Record(s) encoded to binary via `_encode_record`
5. Bytes written to the open file descriptor
6. `_do_sync()` decides whether to fsync based on mode
7. `_maybe_rotate()` checks file size and rotates if needed
8. Lock released, sequence number returned

### Recovery path (constructor)

1. `os.makedirs` ensures the directory exists
2. `_recover_seq_num()` scans every `.wal` file, reads all records, and finds `max(seq_num)` — this is O(total WAL size)
3. `_open_latest()` opens the most recent file for appending if it's under the size limit, otherwise rotates

### Replay path

1. `_fd` is flushed (but not fsynced — just ensuring buffered writes are visible)
2. All WAL files are scanned in sorted order via `_read_all_records()`
3. Records with `seq_num <= after_seq` are skipped
4. Only `PUT`/`DELETE` records are returned; `COMMIT`/`CHECKPOINT` are filtered

### Truncation path

1. Current file is flushed, fsynced, and closed
2. Each WAL file is scanned; records with `seq_num > up_to_seq` are kept
3. Empty files are deleted; files with remaining records are rewritten
4. `_open_latest()` re-opens for future writes

## Invariants

- **Sequence numbers are strictly monotonic.** Every call to `append`, `append_batch`, or `checkpoint` increments `_seq_num` under the lock before writing. No two records share a sequence number.
- **Batch atomicity relies on COMMIT markers.** `append_batch` writes all operations plus a trailing `COMMIT` in a single `write()` call. If the process crashes before the COMMIT is fsynced, `replay()` will still return those records (see below), but a more sophisticated consumer could use COMMIT presence to distinguish committed from uncommitted batches.
- **Replay does NOT enforce COMMIT semantics.** `replay()` returns all `PUT`/`DELETE` records regardless of whether a `COMMIT` follows them. The comment in the code acknowledges this: "All PUT/DELETE records are included; COMMIT/CHECKPOINT are filtered out." A caller needing strict atomicity would need to implement COMMIT-aware filtering externally.
- **CRC mismatch halts replay.** `_read_all_records` catches `ValueError` from `_read_record` and stops iteration entirely (`return`, not `continue`). Corruption in any file aborts reading of all subsequent files.
- **Truncation rewrites files in place.** This is not crash-safe — if the process crashes during `truncate()`, a partially-rewritten file could leave the WAL in an inconsistent state. A production implementation would write to a temp file and atomic-rename.

## Error Handling

- **CRC mismatch** → `ValueError` raised by `_read_record`. Caught by `_read_all_records` which terminates iteration (records after corruption are lost).
- **Partial reads (torn writes)** → `_read_record` returns `None`, treated as EOF. This is the correct behavior for crash recovery — a partial trailing record is simply the last incomplete write before the crash.
- **No explicit error handling** for disk-full, permission errors, or I/O failures during `write()`/`fsync()`. These would propagate as `OSError` to the caller.
- **`truncate()` is not atomic.** A crash during truncation can corrupt the WAL. This is a known limitation of the reference implementation.

## Topics to Explore

- [file] `write-ahead-log/test_wal.py` — Test cases reveal the expected API contract and edge cases (batch semantics, replay filtering, rotation behavior)
- [function] `write-ahead-log/wal.py:_encode_record` — The binary format is the durability contract; understanding the exact byte layout is essential for debugging corrupt WAL files
- [general] `wal-commit-semantics` — The replay method ignores COMMIT markers — investigate whether this is intentional simplification or a bug, and what the implications are for batch atomicity guarantees
- [file] `log-structured-merge-tree/lsm.py` — LSM trees typically use a WAL for their memtable; compare how this WAL would integrate with the LSM implementation in the same repo
- [general] `truncation-crash-safety` — The current truncate rewrites files in place without atomic rename; explore what a crash-safe truncation would look like (write-to-temp + fsync + rename)

## Beliefs

- `wal-replay-ignores-commit` — `replay()` returns all PUT/DELETE records regardless of whether they are followed by a COMMIT record, meaning uncommitted batches are replayed identically to committed ones
- `wal-crc-does-not-cover-seqnum` — The CRC32 checksum covers only `op_type + key + value`, so a corrupted sequence number would not be detected by integrity checking
- `wal-truncate-not-crash-safe` — `truncate()` rewrites WAL files in place without atomic rename, so a crash during truncation can leave the log in an inconsistent state
- `wal-corruption-halts-all-replay` — A CRC mismatch in any WAL file causes `_read_all_records` to stop iteration entirely, silently dropping valid records in subsequent files
- `wal-batch-sync-always-forced` — `append_batch` calls `_do_sync(force=True)`, bypassing the batch sync counter to ensure the COMMIT record is durable regardless of sync mode

