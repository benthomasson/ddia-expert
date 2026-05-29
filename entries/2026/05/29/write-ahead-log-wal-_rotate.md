# Function: _rotate in write-ahead-log/wal.py

**Date:** 2026-05-29
**Time:** 06:36

# `WriteAheadLog._rotate`

## Purpose

`_rotate` closes the current WAL segment file and opens a fresh one with a sequentially numbered filename. WAL implementations split their log across multiple files (segments) so that old, fully-checkpointed segments can be deleted without rewriting the active file. This method is the mechanism that creates the next segment in that chain.

## Contract

**Preconditions:**
- `self._dir` exists on disk (guaranteed by `__init__` calling `os.makedirs`).
- The caller holds `self._lock` (every call site — `_open_latest`, `_maybe_rotate`, and indirectly `append`/`append_batch`/`checkpoint`/`truncate` — acquires the lock before reaching `_rotate`).

**Postconditions:**
- Any previously open file descriptor is flushed, fsynced, and closed.
- `self._current_file` points to a new path of the form `{dir}/{NNNNNN}.wal`.
- `self._fd` is an open file descriptor in append-binary mode to that new file.
- The new file exists on disk (created by `open(..., "ab")`).

**Invariants:**
- WAL filenames are zero-padded 6-digit integers, monotonically increasing.
- At most one WAL file is open for writing at any time.

## Parameters

None — this is a private method operating entirely on instance state.

## Return Value

None. All effects are via mutation of `self._fd` and `self._current_file`.

## Algorithm

1. **Close the old segment.** If `self._fd` is not `None`, flush the userspace buffer, call `os.fsync` to force the kernel buffer to durable storage, then close the file descriptor. This ensures no data from the prior segment is lost.

2. **Determine the next filename.** List all existing `.wal` files (sorted lexicographically, which equals numerically due to zero-padding). If any exist, parse the highest-numbered filename by splitting on `"."` and taking the integer prefix, then add 1. If no files exist, start at 1.

3. **Open the new segment.** Construct the path as `{next_num:06d}.wal` (e.g., `000001.wal`, `000002.wal`) and open it in append-binary mode. Store both the path and the fd on the instance.

## Side Effects

- **Disk I/O:** `fsync` on the old file, creation and `open` of a new file.
- **State mutation:** `self._fd` and `self._current_file` are replaced.
- **File descriptor lifecycle:** The old fd is closed; a new one is opened. If the caller doesn't eventually close the WAL, the new fd leaks.

## Error Handling

No exceptions are caught. If `os.fsync` or `open` fails (e.g., disk full, permission denied), the exception propagates to the caller. This is appropriate — a WAL that can't write is a fatal condition.

There's one subtle assumption: if `self._fd` is non-None but already closed (e.g., due to a bug elsewhere), calling `self._fd.flush()` will raise `ValueError`. The code trusts that `_fd` is either `None` or a valid open descriptor.

## Usage Patterns

`_rotate` is called from two places:

1. **`_open_latest`** — at startup, when no WAL files exist or the latest one is already at capacity. This is the "create the very first segment" path.
2. **`_maybe_rotate`** — after each write operation, when `self._fd.tell() >= self._max_file_size`. This enforces the segment size limit.

Callers never call `_rotate` directly from outside the class (it's private by convention).

## Dependencies

- `os.fsync`, `os.path.basename`, `os.path.join`, `os.listdir` — filesystem operations.
- `self._wal_files()` — returns the sorted list of existing segment paths. The filename-parsing logic in `_rotate` must stay consistent with the naming scheme `_wal_files` expects (files ending in `.wal`, sorted lexicographically).

## Assumptions Not Enforced by Types

- **No concurrent directory access.** If another process creates `.wal` files in the same directory, `_rotate` could collide on filenames. The threading lock only protects in-process concurrency.
- **Filename format is rigid.** `_rotate` assumes every `.wal` file in the directory follows the `NNNNNN.wal` convention. A file named `notes.wal` would cause `int(...)` to raise `ValueError`.
- **`_fd` truthiness equals open-ness.** The `if self._fd` guard assumes a non-None fd is still open. There's no explicit check for `fd.closed`.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_maybe_rotate` — The trigger that decides when rotation happens based on file size
- [function] `write-ahead-log/wal.py:_open_latest` — Startup logic that decides between resuming the last segment and rotating
- [function] `write-ahead-log/wal.py:truncate` — How old segments are garbage-collected after checkpointing, which interacts with the segment numbering scheme
- [general] `wal-segment-sizing-tradeoffs` — How `max_file_size` affects recovery time (more segments = faster truncation but more files to scan on replay) and write amplification
- [file] `write-ahead-log/test_wal.py` — Test coverage for rotation edge cases: empty directories, size-boundary writes, recovery after rotation

## Beliefs

- `wal-rotate-fsync-before-close` — `_rotate` always fsyncs the outgoing segment before closing it, ensuring no buffered writes are lost during rotation
- `wal-segment-naming-sequential` — WAL segment filenames are zero-padded 6-digit integers (`000001.wal`, `000002.wal`, ...) derived from the highest existing filename plus one
- `wal-single-writer-fd` — At most one file descriptor is open for WAL writes at any time; `_rotate` closes the old before opening the new
- `wal-rotate-no-gap-protection` — If a segment file is manually deleted from the directory, `_rotate` can produce a filename that reuses a previously-used number, since it only inspects the current highest filename

