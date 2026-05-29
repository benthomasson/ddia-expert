# Function: _open_latest in write-ahead-log/wal.py

**Date:** 2026-05-29
**Time:** 08:26

# `WriteAheadLog._open_latest`

## Purpose

`_open_latest` resumes writing to the most recent WAL file if it still has room, or triggers creation of a new one. It's the startup/recovery path that ensures `self._fd` and `self._current_file` point to a valid, writable WAL file after the object is constructed or after `truncate()` rewrites the log.

Without this method, the WAL would either always create a fresh file on startup (wasting space and fragmenting the log) or require the caller to decide which file to append to.

## Contract

**Preconditions:**
- `self._dir` exists (guaranteed by `os.makedirs` in `__init__`).
- `self._max_file_size` is a positive integer.
- `self._fd` may be `None` (true at init and after `truncate` closes it).

**Postconditions:**
- `self._fd` is an open file descriptor in binary-append mode.
- `self._current_file` is the path to that file.
- The file is either the last existing `.wal` file (if under the size cap) or a freshly created one.

**Invariant:** After this method returns, the WAL is ready to accept `write()` calls on `self._fd`.

## Parameters

None — this is a private method operating entirely on instance state.

## Return Value

None. All output is through side effects on `self._fd` and `self._current_file`.

## Algorithm

1. **List existing WAL files** — calls `_wal_files()`, which returns lexicographically sorted `.wal` paths. Because filenames are zero-padded integers (`000001.wal`), lexicographic order equals chronological order.

2. **Check if the latest file has room** — if any files exist, it takes the last one and checks its on-disk size via `os.path.getsize`. If strictly less than `_max_file_size`, it opens that file in binary-append mode (`"ab"`) and returns immediately.

3. **Otherwise, rotate** — if no files exist or the latest file is at/above the size cap, it delegates to `_rotate()`, which creates a new numbered WAL file and opens it.

The key decision is the `<` comparison: a file exactly at `_max_file_size` triggers rotation. This means the last write that pushed the file to or past the limit will land in the old file (written by `_maybe_rotate` on the *next* call), and the next startup will correctly rotate.

## Side Effects

- Opens a file descriptor (`self._fd`) — the caller is responsible for eventually closing it (via `close()` or another `_rotate()`).
- Sets `self._current_file` to the path of the active WAL file.
- If `_rotate()` is called, it may flush/fsync/close a previous `self._fd` (though at init time, `_fd` is `None` so `_rotate` skips that branch).

## Error Handling

No explicit error handling. The method can raise:
- **`OSError`/`PermissionError`** from `os.path.getsize()` if the file was deleted between `_wal_files()` and the size check (TOCTOU race).
- **`OSError`** from `open(..., "ab")` if the file is inaccessible.
- Any exception from `_rotate()` propagates unchanged.

None of these are caught — a failure here is fatal to WAL construction, which is the correct behavior (you don't want a half-initialized WAL silently accepting writes).

## Usage Patterns

Called in exactly two places:

1. **`__init__`** — ensures the WAL is immediately writable after construction.
2. **`truncate`** — after rewriting/removing WAL files, re-opens the active file so subsequent appends work.

Both callers expect that after `_open_latest()`, they can call `self._fd.write()` without further setup. This method is never called under `self._lock`  directly — in `__init__` there's no contention yet; in `truncate`, the lock is already held by the caller.

## Dependencies

- **`_wal_files()`** — provides the sorted list of WAL file paths.
- **`_rotate()`** — handles new-file creation, including flushing/closing the old file and numbering the new one.
- **`os.path.getsize()`** — used to check on-disk size rather than an in-memory counter, which means it reflects the true file size even after a crash/restart.

## Assumptions Not Enforced by Types

- WAL filenames are always zero-padded integers (`000001.wal`), so lexicographic sort equals numeric sort. If someone manually creates `foo.wal` in the directory, `_wal_files()` will include it and `_rotate()` will crash on `int(...)`.
- No other process is concurrently writing to or deleting files in `self._dir` between the `_wal_files()` call and the `open()`.
- `_max_file_size` is positive. A zero or negative value would cause every call to fall through to `_rotate()`, creating a new file each time.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_rotate` — The counterpart that creates new WAL files; understanding the naming scheme and fsync discipline here completes the picture of file lifecycle management.
- [function] `write-ahead-log/wal.py:truncate` — The only other caller of `_open_latest`; shows how the WAL rewrites itself after a checkpoint and must re-establish a writable file.
- [function] `write-ahead-log/wal.py:_maybe_rotate` — Called after every write; its size check (`_fd.tell()`) differs from `_open_latest`'s (`os.path.getsize`) — worth understanding why.
- [general] `wal-size-check-toctou` — The gap between `_wal_files()` and `os.path.getsize()` is a TOCTOU window; explore whether this matters in practice given the single-writer design.
- [file] `write-ahead-log/test_wal.py` — Tests for rotation, truncation, and recovery that exercise `_open_latest` indirectly.

## Beliefs

- `wal-open-latest-postcondition` — After `_open_latest` returns, `self._fd` is a valid open file descriptor in append mode and `self._current_file` is its path.
- `wal-size-check-uses-disk` — `_open_latest` checks on-disk file size via `os.path.getsize`, not an in-memory counter, making it correct across crash/restart boundaries.
- `wal-exactly-at-limit-rotates` — A WAL file whose size equals `_max_file_size` triggers rotation; only files strictly smaller are reused.
- `wal-open-latest-two-callers` — `_open_latest` is called only from `__init__` and `truncate`, both of which require a writable WAL file as a postcondition.

