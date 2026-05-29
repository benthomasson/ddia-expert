# Function: _read_all_records in write-ahead-log/wal.py

**Date:** 2026-05-29
**Time:** 06:30

## `_read_all_records` — WAL Record Iterator

### Purpose

`_read_all_records` is the internal read path for the WAL. It iterates over every WAL segment file in sequence order, deserializes each binary record, and yields them one at a time. It exists to provide a single stream abstraction over what is physically a set of numbered `.wal` files on disk.

It's the foundation for two public operations: `replay()` (crash recovery) and `iterate()` (raw inspection).

### Contract

- **Precondition**: The caller is responsible for flushing the write file descriptor before calling. Both `replay()` and `iterate()` do this explicitly (`self._fd.flush()`). If you skip the flush, the iterator may miss records still in the userspace buffer.
- **Postcondition**: Every yielded record has a valid CRC — `_read_record` verified it. The caller never sees a corrupt record.
- **Invariant**: Records are yielded in file order (lexicographic filename sort via `_wal_files()`), then in byte offset order within each file. This matches write order because filenames are zero-padded monotonic integers (`000001.wal`, `000002.wal`, ...) and records are appended sequentially.

### Parameters

None — it reads `self._dir` implicitly via `self._wal_files()`.

### Return Value

`Iterator[WALRecord]` — a lazy generator. Records include all operation types (PUT, DELETE, COMMIT, CHECKPOINT). The caller is responsible for filtering; `replay()` filters to PUT/DELETE, for example.

The iterator terminates in one of two ways:
1. **Clean exhaustion** — all files read, all records yielded.
2. **Early termination on corruption** — a CRC mismatch in any file stops the entire iterator immediately, even if later files are clean.

### Algorithm

```
for each WAL file (sorted by filename):
    open the file in binary mode
    loop:
        try to read one record
        if EOF or partial record → break (move to next file)
        if valid → yield it
        if CRC mismatch → return (stop everything)
```

The key distinction is `break` vs `return`:

- `break` on `None` (EOF): moves to the **next file**. A cleanly-written file that ends at a record boundary produces `None` from `_read_record`.
- `return` on `ValueError` (corruption): **terminates the entire generator**. No more records from any file will be yielded.

This is a deliberate design choice. A corrupt record means the write was interrupted (crash, power loss). Any records after the corruption point — including in subsequent files — cannot be trusted because they may depend on state that was never committed. This is the standard WAL recovery rule: stop at the first sign of damage.

### Side Effects

- **I/O**: Opens and reads every `.wal` file in the log directory. Files are opened read-only (`"rb"`) and closed deterministically via `with`.
- **No mutation**: Does not modify any files or internal state. Pure read operation.

### Error Handling

| Source | Error | Behavior |
|--------|-------|----------|
| `_read_record` returns `None` | EOF / partial read | `break` — advance to next file |
| `_read_record` raises `ValueError` | CRC mismatch | `return` — stop the entire iterator |
| File doesn't exist / permission error | `OSError` | **Not caught** — propagates to caller |

The `ValueError` swallowing is intentional and correct for crash recovery: corruption marks the end of the reliable log. But note that filesystem errors (missing files, permission denied) are *not* caught — those indicate operational problems, not crash recovery scenarios.

### Usage Patterns

Called in two places:

1. **`replay(after_seq)`** — filters to PUT/DELETE records past a given sequence number. This is the crash recovery path.
2. **`iterate()`** — yields all raw records including COMMIT and CHECKPOINT. Used for inspection/debugging.

Both callers flush the write fd first. `_read_all_records` does not acquire `self._lock` — the callers handle locking where needed (`replay` doesn't hold the lock during iteration; `iterate` releases it after flushing).

### Dependencies

- **`self._wal_files()`** — returns sorted file paths. The sort order is critical; wrong order means wrong replay.
- **`_read_record(f)`** — module-level function that handles binary deserialization and CRC validation. This is where the actual parsing and integrity checking lives.

### Assumptions Not Enforced by Types

1. **WAL files are not modified concurrently** by another process during iteration. There's no file locking.
2. **`_wal_files()` returns files in write order.** This relies on filenames being zero-padded integers. If someone manually creates `foo.wal`, it would sort unpredictably.
3. **The write fd has been flushed.** The method itself doesn't flush — it trusts the caller.
4. **Corruption only happens at the tail.** The stop-on-first-corruption strategy assumes damage is from an interrupted write at the end, not a bit-flip in the middle of an otherwise valid file. A mid-file bit-flip would silently discard all subsequent valid records.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_record` — The binary deserialization and CRC validation that `_read_all_records` delegates to; understanding the wire format is essential
- [function] `write-ahead-log/wal.py:replay` — The primary consumer; shows how raw records are filtered into a crash-safe replay sequence
- [function] `write-ahead-log/wal.py:_recover_seq_num` — Uses a similar read loop but with `break` instead of `return` on corruption — compare the two strategies and consider why they differ
- [function] `write-ahead-log/wal.py:truncate` — The write-side counterpart; removes old records and rewrites files, which changes what `_read_all_records` will see on next call
- [general] `wal-corruption-models` — The difference between tail corruption (interrupted append) vs mid-file corruption (bit rot) and how production WALs like SQLite/RocksDB handle each case

---

## Beliefs

- `wal-corruption-stops-all-files` — A CRC mismatch in any WAL file terminates the entire `_read_all_records` iterator, discarding valid records in subsequent files
- `wal-eof-advances-file` — An EOF or partial read within a single WAL file causes the iterator to move to the next file, not terminate
- `wal-read-all-requires-caller-flush` — `_read_all_records` does not flush the write fd; both callers (`replay`, `iterate`) are responsible for flushing before calling
- `wal-read-all-no-lock` — `_read_all_records` does not acquire `self._lock`; thread safety is the caller's responsibility
- `wal-recover-seq-vs-read-all-differ` — `_recover_seq_num` uses `break` on corruption (continues to next file) while `_read_all_records` uses `return` (stops everything) — recovery is lenient, replay is strict

