# Function: _maybe_rotate in write-ahead-log/wal.py

**Date:** 2026-05-29
**Time:** 08:25

## `_maybe_rotate` — Conditional WAL File Rotation

### Purpose

`_maybe_rotate` is a size-based rotation guard. It checks whether the current WAL segment file has grown past the configured size limit (`_max_file_size`, default 10 MB) and, if so, triggers rotation to a new file. This prevents any single WAL file from growing unbounded, which matters for two reasons: it bounds the cost of replaying a single segment during crash recovery, and it lets `truncate()` delete entire files worth of old records rather than rewriting one monolithic file.

### Contract

- **Precondition**: Must be called under `self._lock`. Every call site (`append`, `append_batch`, `checkpoint`) already holds the lock.
- **Postcondition**: After returning, `self._fd` points to a file whose current position is below `_max_file_size` — unless a single write exceeded the limit in one shot (more on this below).
- **Invariant**: `self._fd` is never `None` after this method completes (assuming it wasn't `None` on entry). The `_rotate()` call always opens a new file descriptor.

### Parameters

None. It reads instance state only: `self._fd` (the open file handle) and `self._max_file_size` (the rotation threshold).

### Return Value

`None`. The method communicates entirely through side effects on instance state.

### Algorithm

1. **Guard on `self._fd`**: If the file descriptor is `None` (e.g., after `close()` or during `truncate()`), do nothing. This is a defensive check — in normal operation `_fd` is always set when this is called.
2. **Check file position**: `self._fd.tell()` returns the current write offset — effectively the file size since the file is opened in append mode (`"ab"`).
3. **Compare against threshold**: If the position is at or above `_max_file_size`, delegate to `self._rotate()`.

That's it — three lines doing one thing.

### Side Effects

When rotation triggers:
- The current file is flushed and fsynced (durability guarantee before closing).
- The current file descriptor is closed.
- A new sequentially-numbered `.wal` file is created and opened for append.
- `self._current_file` and `self._fd` are updated to point to the new segment.

When rotation does not trigger: no side effects at all.

### Error Handling

None explicit. If `tell()` fails (e.g., on a closed fd), the exception propagates to the caller. If `_rotate()` fails to create the new file (permissions, disk full), that also propagates uncaught. The method trusts that `_fd` is in a valid state when called.

### Usage Patterns

Called as the final step in every write path — after `_do_sync()`, never before:

```python
# In append():
self._fd.write(data)
self._do_sync()
self._maybe_rotate()    # rotate AFTER the write is durable

# Same pattern in append_batch() and checkpoint()
```

The ordering is deliberate: the write and sync happen to the current file first, so the data is durable before rotation closes that file. If rotation were called before sync, the fsync inside `_rotate()` would still cover it (since `_rotate` flushes before closing), but the current ordering makes the intent clearer and keeps `_do_sync`'s batch-counting logic coherent.

**Caller obligation**: callers must hold `self._lock` before calling this method. There is no internal locking.

### Dependencies

- `self._rotate()` — does the actual file creation and descriptor swap.
- `self._fd.tell()` — Python file object's position query. Works correctly with `"ab"` mode on POSIX: the OS sets the position to end-of-file on each write, so `tell()` reflects the true file size.

### Assumptions Not Enforced by the Type System

1. **A single write can exceed `_max_file_size`**. If a batch is larger than the limit, `_maybe_rotate` will rotate *after* that write, meaning one segment can temporarily exceed the cap. The check is `>=`, not a pre-write check, so this is by design — it's a soft limit.
2. **`_fd` opened in append mode**. `tell()` only equals file size if the file was opened with `"ab"`. If someone changed the open mode, `tell()` could report a misleading position.
3. **Called under lock**. Nothing prevents a direct call without the lock — the method doesn't acquire it and doesn't assert it's held.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_rotate` — The actual rotation logic: how segment numbering works, the flush-fsync-close sequence, and what happens when no prior files exist.
- [function] `write-ahead-log/wal.py:_do_sync` — The durability strategy that runs immediately before `_maybe_rotate` — understanding sync modes (sync vs batch vs none) explains why rotation can safely close the file.
- [function] `write-ahead-log/wal.py:truncate` — The other mechanism that removes WAL segments; understanding how truncation and rotation interact (truncation calls `_open_latest`, which may itself call `_rotate`).
- [general] `wal-segment-sizing-tradeoffs` — How `_max_file_size` affects recovery time, truncation granularity, and disk usage. Smaller segments mean faster per-file replay but more files to manage.
- [file] `write-ahead-log/test_wal.py` — Test cases that exercise rotation behavior, particularly edge cases around writes that exceed the size limit.

---

## Beliefs

- `wal-rotation-is-post-write` — Rotation is checked after each write is synced, never before; a single write can produce a segment larger than `_max_file_size`.
- `wal-max-file-size-is-soft-limit` — `_max_file_size` is a soft cap: the check triggers rotation for the *next* write, it does not prevent the current write from exceeding the limit.
- `maybe-rotate-requires-lock` — `_maybe_rotate` must be called under `self._lock` but does not acquire or assert the lock itself; all three call sites (`append`, `append_batch`, `checkpoint`) satisfy this obligation.
- `rotation-preserves-fd-invariant` — After `_maybe_rotate` returns, `self._fd` is guaranteed non-None and open for writing (assuming it was non-None on entry), because `_rotate()` always opens a new file.

