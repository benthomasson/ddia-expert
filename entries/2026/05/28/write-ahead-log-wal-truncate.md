# Function: truncate in write-ahead-log/wal.py

**Date:** 2026-05-28
**Time:** 18:46

# `WriteAheadLog.truncate`

## Purpose

`truncate` is the WAL's garbage collection mechanism. After a database checkpoint (where WAL contents have been flushed to the main data file), records that are now durable elsewhere no longer need crash-recovery protection. This method removes those records, reclaiming disk space and keeping replay times short.

In DDIA terms, this is the "log compaction" step — without it, the WAL grows unbounded.

## Contract

- **Precondition**: `up_to_seq` should be a sequence number that the caller knows is safely persisted elsewhere (e.g., after a successful checkpoint/compaction of the main data store). Passing a sequence number for records that haven't been persisted elsewhere means those records are gone forever.
- **Postcondition**: No record with `seq_num <= up_to_seq` remains in any WAL file. WAL files that become empty are deleted entirely. The WAL is ready for new appends.
- **Invariant**: The lock is held for the entire operation — no concurrent reads or writes can interleave.

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `up_to_seq` | `int` | Inclusive upper bound. Every record with a sequence number at or below this value is discarded. Passing `0` is a no-op (no records have seq 0). Passing `current_seq_num()` deletes everything. |

There's no validation that `up_to_seq` is sane — passing a value beyond `current_seq_num()` silently deletes all records without error.

## Return Value

`None`. Success is silent. The caller infers correctness by the absence of exceptions.

## Algorithm

**Step 1 — Close the active file descriptor.**
Flush pending writes and fsync to ensure every buffered record hits disk before we start rewriting files. The fd is set to `None` so the rewrite loop doesn't conflict with the open append handle.

**Step 2 — Iterate every `.wal` file in sorted order.**
For each file:
1. Read all records, collecting those with `seq_num > up_to_seq` into `kept`.
2. If `kept` is empty, the entire file is obsolete — delete it with `os.remove`.
3. If `kept` is non-empty, **rewrite the file in place** (`"wb"` mode) with only the surviving records, then flush + fsync to ensure durability.

**Step 3 — Reopen for appending.**
`_open_latest()` either reopens the last surviving file (if under `max_file_size`) or creates a new one via `_rotate()`.

The key detail: this is **not** an atomic operation across files. If the process crashes mid-truncate, some files may be fully truncated while others are untouched. However, the data is still consistent — records that survived the crash are still valid, and the next truncate call will finish the job.

## Side Effects

- **Disk I/O**: Reads, rewrites, and potentially deletes multiple WAL files. Each surviving file gets an `fsync`.
- **File descriptor lifecycle**: Closes the current fd before rewriting, then reopens via `_open_latest()`. Any external reference to `self._fd` would break (though the lock prevents concurrent access).
- **State mutation**: `self._fd` and `self._current_file` are both updated as a consequence of `_open_latest()`.

## Error Handling

- **CRC mismatches / corrupt records**: `_read_record` raises `ValueError` on CRC failure. The `except ValueError: break` stops reading that file — records after the corruption point are silently lost, even if they have `seq_num > up_to_seq`. This is a deliberate design choice: corruption marks the end of trustworthy data.
- **Filesystem errors**: `OSError` from file operations (`open`, `os.remove`, `os.fsync`) propagate uncaught. If the process crashes after deleting some files but before rewriting others, the WAL is in a partially truncated state — recoverable but potentially missing the records from deleted files.
- **No rollback**: If rewriting a file fails mid-write (disk full, permissions), the file may be left truncated or empty. There's no temp-file-then-rename pattern here.

## Usage Patterns

Typical call site:

```python
# After successfully checkpointing the database state
checkpoint_seq = wal.checkpoint()
# ... flush main data store to disk ...
wal.truncate(checkpoint_seq)
```

Caller obligations:
1. **Only truncate what's durable elsewhere.** This is the caller's responsibility — `truncate` doesn't verify that records have been applied.
2. **Don't call concurrently with append.** The internal lock handles this, but callers should understand that truncate blocks all writes for its duration, which can be significant for large WALs.

## Dependencies

| Dependency | Usage |
|------------|-------|
| `os` | File operations: `remove`, `fsync`, `makedirs` |
| `struct` | Binary record encoding via `_encode_record` |
| `zlib` | CRC32 checksums inside `_encode_record` / `_read_record` |
| `threading.Lock` | Mutual exclusion with `append`, `replay`, etc. |
| `_read_record` | Module-level function for deserializing binary WAL records |
| `_encode_record` | Module-level function for serializing records back to binary |
| `_open_latest` | Instance method to resume appending after rewrite |
| `_wal_files` | Instance method returning sorted list of WAL file paths |

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:replay` — The read-side counterpart to truncate; understanding both clarifies the full WAL lifecycle
- [function] `write-ahead-log/wal.py:_encode_record` — The binary format that truncate must faithfully reproduce when rewriting files
- [general] `crash-safety-of-truncate` — The lack of atomic rename means a crash mid-truncate can lose records; compare with how real databases (e.g., SQLite, PostgreSQL) handle WAL truncation using temp files or header flags
- [file] `write-ahead-log/test_wal.py` — Test cases for truncate reveal the expected behavior around edge cases like truncating past the end or truncating with concurrent appends
- [general] `log-structured-checkpoint-coordination` — How the checkpoint sequence number flows between the main data store and the WAL to ensure truncate is safe

## Beliefs

- `wal-truncate-inclusive-boundary` — `truncate(n)` removes records with `seq_num <= n` (inclusive), keeping only those strictly greater than `n`
- `wal-truncate-drops-after-corruption` — If a WAL file contains a corrupt record, all records after that point in the file are silently discarded during truncate, regardless of their sequence number
- `wal-truncate-not-atomic` — Truncation rewrites files in place without a temp-file-rename pattern; a crash mid-truncate can leave the WAL in a partially truncated state
- `wal-truncate-deletes-empty-files` — WAL files where every record has `seq_num <= up_to_seq` are deleted entirely rather than left as empty files
- `wal-truncate-blocks-all-operations` — The method holds `self._lock` for its entire duration, blocking all concurrent appends, replays, and iterates

