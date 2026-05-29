# Function: truncate in log-structured-merge-tree/lsm.py

**Date:** 2026-05-29
**Time:** 08:03



## `WAL.truncate` — Write-Ahead Log Reset

### Purpose

Clears the WAL file so it contains zero bytes, then re-opens it in append mode for future writes. This is called after the memtable has been successfully flushed to an SSTable on disk — at that point the WAL's recovery data is no longer needed, because the data it protected is now durable in the SSTable.

Without truncation, the WAL would replay stale entries on the next startup, re-inserting data that's already persisted in SSTables.

### Contract

- **Precondition**: `self._fd` is an open file handle (the constructor guarantees this).
- **Postcondition**: The WAL file at `self._path` exists and is empty (0 bytes). `self._fd` is a valid file handle open in append mode, ready for new `append()` calls.
- **Invariant**: After truncate returns, the object is in the same logical state as a freshly constructed WAL — empty file, append-mode handle — but reuses the same path.

### Parameters

None (operates on instance state).

### Return Value

None.

### Algorithm

1. **Close the current handle** — `self._fd.close()`. Flushes any OS-buffered data and releases the file descriptor.
2. **Open in write-binary mode (`"wb"`)** — This is the key step. Opening with `"wb"` truncates the file to zero length per POSIX/Python semantics. The file now exists but is empty.
3. **Close the write handle** — The truncation is committed to the filesystem.
4. **Re-open in append-binary mode (`"ab"`)** — Restores the handle to append mode so subsequent `append()` calls work correctly. Append mode guarantees all writes go to the end of file, which is critical for WAL correctness.

The two-open dance (`wb` then `ab`) exists because Python's `open("wb")` truncates on open but positions the cursor at offset 0 for overwrite — not safe for a WAL. The second open in `"ab"` ensures the invariant that all future writes append.

### Side Effects

- **File I/O**: The WAL file is truncated to 0 bytes. This is destructive and irreversible.
- **File descriptor churn**: Three `open`/`close` cycles happen. This is fine for correctness but wouldn't be ideal in a hot path (it's called once per flush, so it's not).
- **State mutation**: `self._fd` is replaced with a new file handle.

### Error Handling

None. If any `open()` or `close()` call raises an `OSError` (disk full, permission denied, file deleted), the exception propagates uncaught. This can leave the object in a partially broken state — e.g., if the second `open` fails, `self._fd` holds a closed handle from step 3, and subsequent `append()` calls will raise `ValueError: I/O operation on closed file`.

### Usage Patterns

Called exactly once per flush cycle, from `LSMTree._flush()`:

```python
def _flush(self):
    # ... write memtable to SSTable ...
    self._wal.truncate()    # SSTable is durable; WAL data is now redundant
```

The caller must ensure the SSTable write completed successfully before calling truncate. If truncate runs before the SSTable is durable, a crash would lose data — the WAL is gone and the SSTable never landed. The current code does this correctly: `SSTable.write()` completes (including closing the file) before `truncate()` is called.

### Dependencies

- Python built-in `open()` / file object `close()`. No external libraries.
- Relies on POSIX `"wb"` semantics for truncation.

### Assumptions Not Enforced by Types

- The path still points to a valid, writable location on disk.
- No other process or thread holds the file open (no file locking).
- `self._fd` has not already been closed by a prior `close()` call — calling `truncate()` after `close()` will raise on the first `self._fd.close()`.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:_flush` — The only call site; shows the ordering guarantee between SSTable write and WAL truncation
- [function] `log-structured-merge-tree/lsm.py:WAL.replay` — The counterpart to truncate: what happens on startup if truncate was never called
- [general] `wal-crash-safety-gap` — The truncate-then-reopen sequence is not atomic; a crash between steps 2 and 4 leaves a valid but empty WAL with no open handle, which is safe but worth reasoning through
- [function] `log-structured-merge-tree/lsm.py:WAL.append` — Uses the file handle that truncate restores; understanding append's flush semantics completes the durability picture
- [general] `os-fsync-durability` — Neither `truncate` nor `_flush` calls `fsync()`, meaning data could be lost in an OS crash even after the Python-level write completes

---

## Beliefs

- `wal-truncate-clears-file` — `WAL.truncate()` reduces the WAL file to zero bytes by opening it in `"wb"` mode, which truncates on open per POSIX semantics
- `wal-truncate-restores-append-mode` — After truncation, the file handle is re-opened in `"ab"` mode so the WAL can immediately accept new `append()` calls
- `wal-truncate-called-after-flush` — `truncate()` is called by `LSMTree._flush()` only after `SSTable.write()` has completed, preserving the invariant that WAL data is cleared only when it's durable elsewhere
- `wal-truncate-no-error-recovery` — If any `open()` or `close()` call within `truncate()` fails, the exception propagates uncaught and can leave `self._fd` in an unusable state
- `wal-no-fsync` — Neither `WAL.truncate()` nor `WAL.append()` calls `os.fsync()`, so durability depends on OS buffer flush timing rather than being guaranteed at the application level

