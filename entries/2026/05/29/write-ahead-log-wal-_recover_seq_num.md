# Function: _recover_seq_num in write-ahead-log/wal.py

**Date:** 2026-05-29
**Time:** 06:40



# `_recover_seq_num` — WAL Sequence Number Recovery

## Purpose

This method scans every existing WAL file on disk to find the highest sequence number that was ever written. It exists to solve a specific crash-recovery problem: after a restart, the WAL needs to resume issuing sequence numbers that are strictly higher than any previously used. Without this, a restarted WAL could reuse sequence numbers, breaking the monotonicity guarantee that downstream consumers (replay, truncation) depend on.

It's called exactly once, during `__init__`, before any new writes occur.

## Contract

- **Precondition**: `self._dir` exists and `self._wal_files()` is functional (the directory was already created by `os.makedirs` earlier in `__init__`).
- **Postcondition**: Returns an integer ≥ 0 representing the highest sequence number found across all WAL files. If no WAL files exist or all are empty, returns 0.
- **Invariant it establishes**: After `__init__` sets `self._seq_num = self._recover_seq_num()`, every subsequent `append`/`append_batch`/`checkpoint` call increments `_seq_num` before writing, so new records always get sequence numbers strictly greater than anything on disk.

## Parameters

None (beyond `self`). All input comes from the filesystem via `self._wal_files()`.

## Return Value

An `int` — the maximum sequence number found, or `0` if no valid records exist. The caller (`__init__`) assigns this directly to `self._seq_num`. Because `append` does `self._seq_num += 1` *before* writing, the first new record will be `max_seq + 1`.

## Algorithm

1. Initialize `max_seq = 0`.
2. Get the sorted list of `.wal` files from the log directory.
3. For each file, open it in binary read mode.
4. Read records one at a time via `_read_record(f)`:
   - If `_read_record` returns `None` → EOF or partial/truncated record at the tail. Move to the next file.
   - If `_read_record` raises `ValueError` → CRC mismatch (corruption). **Stop scanning this file entirely** and move to the next.
   - Otherwise, update `max_seq` if this record's `seq_num` is higher.
5. After all files are scanned, return `max_seq`.

## Side Effects

Read-only I/O against the WAL directory. No mutations to disk or object state — this is a pure scan that returns a value.

## Error Handling

There are two failure modes in `_read_record`, and this method handles them differently:

| Condition | `_read_record` behavior | `_recover_seq_num` response |
|---|---|---|
| EOF / truncated record | Returns `None` | Breaks inner loop, continues to next file |
| CRC mismatch | Raises `ValueError` | Breaks inner loop, continues to next file |

This is notably more forgiving than `_read_all_records`, which **returns entirely** (stops all files) on a `ValueError`. The recovery scan treats corruption as file-local: it abandons the corrupted file but still reads subsequent files. This makes sense — during recovery you want to find the highest sequence number across the entire WAL, even if one file has a corrupted tail.

**Assumption**: any `OSError` from `open()` or `f.read()` (permissions, missing file) is *not* caught and will propagate up to the caller, aborting initialization.

## Usage Patterns

Called once in `WriteAheadLog.__init__`:

```python
self._seq_num = 0
# ...
self._seq_num = self._recover_seq_num()
```

The initial `self._seq_num = 0` is immediately overwritten, so it's effectively dead code — the recovery result is what matters. No other code calls this method.

## Dependencies

- `self._wal_files()` — provides the sorted file list. Sorting is important: while `_recover_seq_num` takes the global max so ordering doesn't affect correctness, the sort order matters for `_open_latest` and `_rotate` which are called immediately after.
- `_read_record(f)` — the module-level binary deserialization function. Handles the wire format parsing and CRC validation.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_all_records` — Compare its corruption handling (stops globally) with `_recover_seq_num` (stops per-file) and understand why they differ
- [function] `write-ahead-log/wal.py:replay` — The primary consumer of sequence numbers; understanding replay shows why monotonic sequence recovery is critical
- [function] `write-ahead-log/wal.py:truncate` — Uses `seq_num` as the GC boundary; see how recovered sequence numbers interact with log compaction
- [general] `wal-crash-recovery-semantics` — How the combination of `_recover_seq_num`, `_open_latest`, and `replay` collectively guarantee no data loss after a crash
- [file] `write-ahead-log/test_wal.py` — Tests that validate recovery behavior, especially around corruption and partial writes

## Beliefs

- `recover-seq-num-is-global-max` — `_recover_seq_num` returns the maximum sequence number across all WAL files, not just the latest file, ensuring correctness even when records are spread across rotated files
- `corruption-handling-is-per-file` — Unlike `_read_all_records` which aborts all scanning on corruption, `_recover_seq_num` treats CRC errors as file-local and continues scanning subsequent files
- `seq-num-zero-means-empty` — A return value of 0 means no valid records exist on disk; the next write will use sequence number 1
- `recover-is-read-only` — `_recover_seq_num` performs no writes or state mutations; it is a pure filesystem scan that returns an integer
- `no-gap-detection` — The method finds the max but does not detect or report gaps in the sequence number space, which could indicate lost records

