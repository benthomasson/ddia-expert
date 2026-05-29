# Function: _do_sync in write-ahead-log/wal.py

**Date:** 2026-05-29
**Time:** 08:30

# `_do_sync` — Write-Ahead Log Fsync Policy

## Purpose

`_do_sync` controls **when buffered WAL writes are durably persisted to disk**. It exists because there's a fundamental tension in WAL design: calling `fsync` after every write guarantees durability but kills throughput; skipping it risks data loss on crash. This method encapsulates the durability policy so callers (`append`, `append_batch`, `checkpoint`) don't need to know which sync strategy is active.

## Contract

**Preconditions:**
- `self._fd` is an open file descriptor in append-binary mode (`"ab"`). No null check is performed — calling this with `_fd = None` will raise `AttributeError`.
- `self._sync_mode` is one of `"sync"`, `"batch"`, or implicitly any other string (which results in no sync at all — a silent "none" mode).

**Postconditions:**
- In `"sync"` mode or when `force=True`: all buffered data is flushed to the OS and `fsync`'d to stable storage. The data survives a process crash, OS crash, or power loss (assuming the storage hardware honors fsync).
- In `"batch"` mode without `force`: the write counter increments. If it hits the threshold, data is fsynced and the counter resets. Otherwise, data remains in the userspace/kernel buffer — **not durable**.
- In any other mode (implicit `"none"`): nothing happens. Data sits in Python's write buffer.

**Invariant:** `self._write_count` is always in `[0, self._batch_sync_count - 1]` after a non-forced call in batch mode.

## Parameters

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `force` | `bool` | `False` | Bypasses the sync mode policy and forces an immediate fsync. Used by `append_batch` and `checkpoint` to guarantee atomicity/durability of critical records regardless of the configured mode. |

**Edge case:** If `force=True` and `_sync_mode == "batch"`, the force path (`_sync_mode == "sync" or force`) takes precedence, but `_write_count` is **not** reset. This means the batch counter keeps accumulating across forced syncs, which is harmless but slightly imprecise — the next batch-triggered sync may come earlier than expected.

## Return Value

None. This is a side-effect-only method.

## Algorithm

```
1. If sync_mode is "sync" OR force is True:
   → flush Python's internal buffer to the OS
   → fsync the file descriptor (block until hardware confirms write)
   → DONE

2. Else if sync_mode is "batch":
   → increment _write_count
   → if _write_count >= _batch_sync_count:
       → flush + fsync (same as above)
       → reset _write_count to 0
   → DONE

3. Otherwise (implicit "none" mode):
   → do nothing — writes remain buffered
```

The two-step `flush()` then `fsync()` is necessary because Python's `io` layer maintains its own buffer separate from the OS page cache. `flush()` pushes data from Python → kernel; `fsync()` pushes data from kernel → disk.

## Side Effects

- **I/O:** Calls `self._fd.flush()` and `os.fsync()`, which may block for milliseconds to seconds depending on the storage device and write queue depth.
- **State mutation:** In batch mode, mutates `self._write_count`. This is the only mode with internal state changes.
- **No locking:** The method itself doesn't acquire `self._lock`. It relies on its callers (`append`, `append_batch`, `checkpoint`) to hold the lock. This is a correctness requirement that isn't enforced by the method signature.

## Error Handling

No exceptions are caught. Both `flush()` and `os.fsync()` can raise `OSError` (disk full, I/O error, bad file descriptor), which will propagate directly to the caller. This is the correct behavior — a failed sync in a WAL is a critical error that the application must handle, not swallow.

## Usage Patterns

Three call sites, each with a different durability need:

```python
# Single record — respects configured policy
def append(self, ...):
    self._fd.write(data)
    self._do_sync()           # might not actually sync in batch mode

# Atomic batch — must be durable
def append_batch(self, ...):
    self._fd.write(bytes(buf))
    self._do_sync(force=True) # always syncs, regardless of mode

# Checkpoint marker — must be durable
def checkpoint(self):
    self._fd.write(...)
    self._do_sync(force=True) # always syncs
```

The pattern is: individual writes tolerate deferred durability (you might lose the last few writes on crash), but batch commits and checkpoints **must** be durable immediately because downstream consumers rely on their presence to determine recovery boundaries.

## Dependencies

- `os.fsync` — POSIX fsync(2) wrapper. On Linux this is a true fsync; on macOS it's `fcntl(F_FULLFSYNC)` only if explicitly called (Python's `os.fsync` maps to `fsync(2)`, which on macOS does **not** guarantee flush to platter unless the drive firmware cooperates). This is a subtle portability assumption.
- `self._fd` — a Python file object opened in `"ab"` mode via the built-in `open()`.

## Assumptions Not Enforced by Types

1. **`_sync_mode` is a free-form string**, not an enum. Passing `"SYNC"` or `"Sync"` silently falls into the no-op branch. There's no validation at construction time either.
2. **`_fd` is never None** when this method is called. The `truncate` method sets `_fd = None` temporarily, and if `_do_sync` were called during that window, it would crash.
3. **Callers hold `self._lock`**. Thread safety depends entirely on caller discipline.
4. **`force=True` in batch mode doesn't reset `_write_count`**, which means the counter drifts. Not a bug, but a subtlety.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:append_batch` — Shows why `force=True` is critical: a batch without a durable COMMIT record could be partially replayed after crash
- [function] `write-ahead-log/wal.py:_rotate` — Also does flush+fsync before closing the old file, forming the other half of the durability story
- [general] `fsync-vs-fdatasync` — `fdatasync` skips metadata updates and is faster; understanding when each is appropriate matters for WAL performance
- [file] `write-ahead-log/test_wal.py` — Check whether tests cover the batch sync counter edge cases and the implicit "none" mode
- [general] `group-commit-optimization` — The batch mode here is a simplified version of the group commit technique used in real databases (PostgreSQL, MySQL) to amortize fsync cost across multiple transactions

## Beliefs

- `wal-sync-mode-unvalidated` — `WriteAheadLog` accepts any string as `sync_mode` without validation; values other than `"sync"` and `"batch"` silently disable all fsync
- `wal-force-sync-skips-batch-counter-reset` — When `_do_sync(force=True)` is called in batch mode, the fsync occurs but `_write_count` is not reset to zero
- `wal-batch-sync-count-controls-durability-window` — In batch mode, up to `batch_sync_count - 1` records may be lost on crash because they haven't been fsynced yet
- `wal-do-sync-requires-caller-lock` — `_do_sync` does not acquire `self._lock`; thread safety depends on every caller holding the lock before invoking it

