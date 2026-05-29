# Function: iterate in write-ahead-log/wal.py

**Date:** 2026-05-28
**Time:** 18:50

# `WriteAheadLog.iterate`

## Purpose

`iterate` provides raw, unfiltered access to every record stored in the WAL — including uncommitted batch operations, COMMIT markers, and CHECKPOINT records. This contrasts with `replay`, which filters out non-data records and skips uncommitted batches. It exists for diagnostic, inspection, or tooling scenarios where the caller needs to see the full on-disk state of the log, not just the logically committed view.

## Contract

- **Precondition**: The WAL instance must be initialized (constructor completed successfully, so `_wal_files()` and `_fd` are in a valid state).
- **Postcondition**: Yields every structurally valid record across all WAL segment files, in file order, stopping at the first corruption (CRC mismatch). No records are skipped or filtered.
- **Invariant**: The method does not modify any WAL state — it is a read-only scan. The flush before iteration ensures that any buffered writes from the current process are visible on disk.

## Parameters

None beyond `self`. The method takes no arguments — it always scans the entire log from the beginning.

## Return Value

Returns an `Iterator[WALRecord]`. Each `WALRecord` contains:
- `seq_num`: monotonically increasing sequence number
- `op_type`: one of `"PUT"`, `"DELETE"`, `"COMMIT"`, `"CHECKPOINT"`, or `"UNKNOWN"`
- `key` / `value`: UTF-8 decoded strings
- `checksum`: the CRC32 stored in the record

The caller must handle:
- An empty iterator (no WAL files or all files are empty)
- Records of **any** op type, including `COMMIT` and `CHECKPOINT`, which `replay` would suppress
- The iterator stopping abruptly at corruption — there is no signal distinguishing "end of log" from "stopped at bad record"

## Algorithm

1. **Acquire lock and flush**: Grabs `_lock`, checks if there's an open file descriptor (`_fd`), and flushes it. This ensures any writes buffered in userspace by the current process are pushed to the OS before reading. The lock is released immediately after the flush — it is **not** held during iteration.

2. **Delegate to `_read_all_records`**: Uses `yield from` to lazily stream records. `_read_all_records` iterates over every `.wal` file in sorted filename order, opens each read-only, and calls `_read_record` in a loop until EOF (`None` return) or CRC failure (`ValueError`).

## Side Effects

- **I/O**: Flushes the write fd (a `flush()` syscall — no `fsync`, so data reaches the OS page cache but isn't necessarily durable). Then opens each WAL segment file for reading during iteration.
- **Lock scope**: The lock is held only during the flush, **not** during the read phase. This means concurrent `append` calls can interleave with iteration. The iterator may or may not see records written after `iterate()` was called, depending on timing and whether those writes land in a file the iterator hasn't opened yet.

## Error Handling

- **CRC mismatch**: `_read_record` raises `ValueError` on CRC failure. `_read_all_records` catches this and **returns** (stops iteration silently). The caller receives all records up to the corruption point with no indication that more data existed.
- **File I/O errors** (missing files, permission errors): Not caught — these propagate as `OSError`/`IOError` to the caller.
- **Partial reads**: If a record is truncated (incomplete write from a crash), `_read_record` returns `None` and iteration moves to the next file.

## Usage Patterns

Typical callers use `iterate` when they need the raw log contents rather than the committed-only view:

```python
wal = WriteAheadLog("/tmp/wal")
for record in wal.iterate():
    print(f"seq={record.seq_num} op={record.op_type} key={record.key}")
```

**Caller obligations**:
- Do not assume all yielded records are committed — partial batches (operations before a missing COMMIT) will appear.
- Do not assume the iterator covers the full log — corruption silently truncates it.
- Be aware that the iterator is **not snapshot-isolated** from concurrent writes.

## Dependencies

- `_read_all_records` (internal): does the actual file scanning and record parsing
- `_wal_files` (internal): returns sorted segment file paths
- `_read_record` (module-level): binary deserialization and CRC verification
- `threading.Lock`: for the flush synchronization
- Standard library: `os`, `struct`, `zlib` (indirectly, through `_read_record`)

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:replay` — Compare with `iterate`: `replay` filters to committed PUT/DELETE records only, showing how the WAL distinguishes raw log access from logical recovery
- [function] `write-ahead-log/wal.py:_read_all_records` — The private generator that does the actual file-by-file, record-by-record scan with corruption-stop semantics
- [function] `write-ahead-log/wal.py:append_batch` — Shows how batched writes use COMMIT records as atomicity boundaries — the records that `iterate` exposes but `replay` consumes
- [general] `wal-concurrency-safety` — The lock is released before iteration begins; explore what guarantees (or lack thereof) exist for concurrent read/write access
- [file] `write-ahead-log/test_wal.py` — Test cases showing expected behavior of iterate vs replay, especially around uncommitted batches and corruption

## Beliefs

- `iterate-yields-all-op-types` — `iterate` yields records of every op type (PUT, DELETE, COMMIT, CHECKPOINT) while `replay` filters to only PUT and DELETE
- `iterate-not-snapshot-isolated` — The lock is released after flush but before iteration begins, so concurrent appends may or may not be visible to the iterator
- `iterate-stops-silently-at-corruption` — When a CRC mismatch is encountered, iteration stops with no exception or sentinel value to the caller
- `iterate-flushes-without-fsync` — Before reading, `iterate` calls `fd.flush()` but not `os.fsync()`, so buffered writes reach the page cache but aren't guaranteed durable on disk

