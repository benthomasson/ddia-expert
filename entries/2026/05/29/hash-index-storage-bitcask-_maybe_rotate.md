# Function: _maybe_rotate in hash-index-storage/bitcask.py

**Date:** 2026-05-29
**Time:** 11:43

## `_maybe_rotate` — Active File Rotation Guard

### Purpose

`_maybe_rotate` enforces the maximum file size invariant in a Bitcask storage engine. Bitcask writes are append-only to a single "active" data file. Without rotation, that file would grow without bound. This method checks whether the active file has reached `max_file_size` and, if so, closes it and opens a fresh one with the next sequential ID.

It exists to keep individual data files bounded in size, which matters for two reasons: (1) compaction operates on immutable (non-active) files, so rotation is what creates compaction candidates, and (2) bounded file sizes keep `mmap`/read operations practical and make crash recovery faster since only the active file needs scanning.

### Contract

- **Precondition**: `self.active_file` is an open file handle in append-binary mode (`"ab"`), and `self.active_file_id` is its corresponding ID. `self.max_file_size` is a positive integer.
- **Postcondition**: If the active file's current write position was `>= max_file_size`, the old file is closed, `active_file_id` is incremented by 1, and a new active file is opened via `_open_active_file()`. The old file becomes immutable (no further writes). If the size was below the threshold, nothing changes.
- **Invariant**: At most one active (writable) file exists at any time. All other data files are immutable.

### Parameters

None — this is a zero-argument instance method. All state is read from `self`.

### Return Value

`None`. This is a guard/side-effect method. The caller doesn't need to handle a return value — the rotation either happened or it didn't.

### Algorithm

1. **Check file position**: `self.active_file.tell()` returns the current write cursor, which equals the file size since the file is opened in append mode.
2. **Compare against threshold**: If the position is `>= self.max_file_size`, proceed with rotation.
3. **Close the current file**: `self.active_file.close()` flushes and closes the write handle. The read handle for this file ID remains open in `self.file_handles` — intentionally, since existing `KeyEntry` records still point to it.
4. **Increment file ID**: `self.active_file_id += 1` picks the next sequential ID. There's no gap detection — it assumes sequential IDs are safe.
5. **Open new active file**: `_open_active_file()` creates a new `.data` file, opens it for appending, and registers a read handle in `self.file_handles`.

### Side Effects

- **File I/O**: Closes one file descriptor, opens two new ones (one `"ab"` for writing, one `"rb"` for reading via `_open_active_file`).
- **State mutation**: Modifies `self.active_file`, `self.active_file_id`, and `self.file_handles`.
- **Disk**: Creates a new (empty) `.data` file on disk.

### Error Handling

None. If `_open_active_file` fails (e.g., permission error, disk full), the exception propagates uncaught. At that point, `self.active_file` has already been closed and `active_file_id` incremented — leaving the store in an inconsistent state. This is a deliberate simplicity tradeoff in a reference implementation: production Bitcask engines (like Riak's) handle this more carefully.

### Usage Patterns

Called as a guard at the top of every mutating operation:

```python
def put(self, key, value):
    self._maybe_rotate()        # ensure room before writing
    offset, size, ts = self._write_record(key, value)
    ...

def delete(self, key):
    self._maybe_rotate()        # tombstones are records too
    self._write_record(key, "")
    ...
```

The caller obligation is simple: call `_maybe_rotate()` **before** `_write_record()`. This means a record can still push the file slightly past `max_file_size` (the check is pre-write, not mid-write), but the next operation will trigger rotation. The file size limit is therefore a soft cap, not a hard one — any individual record can exceed the boundary by up to one record's worth of bytes.

### Dependencies

- `self._open_active_file()` — handles the mechanics of creating and registering the new file.
- `self.active_file` — a Python file object opened in `"ab"` mode, where `.tell()` reflects total bytes written.
- `self.max_file_size` — set once at `__init__`, defaults to 10 MiB.

### Assumptions Not Enforced by Types

1. **`tell()` accuracy on append files**: The code assumes `tell()` on an `"ab"` file returns the file's total size. This relies on the `seek(0, 2)` call in `_open_active_file` and on the OS not buffering in a way that makes `tell()` lag behind actual writes. This works on POSIX but is technically implementation-dependent.
2. **No concurrent writers**: Nothing prevents two threads from both passing the size check and writing simultaneously. Bitcask's original design assumes a single writer process.
3. **Sequential ID availability**: The method blindly increments `active_file_id` without checking whether a file with that ID already exists on disk. After compaction (which renumbers files), this could collide — though `compact()` updates `active_file_id` to avoid this.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:_open_active_file` — The companion that actually creates the new file and registers read/write handles
- [function] `hash-index-storage/bitcask.py:compact` — Compaction is the downstream consumer of rotation: it merges the immutable files that rotation produces
- [general] `bitcask-soft-vs-hard-size-limits` — The pre-write rotation check means `max_file_size` is a soft cap; explore what a record-splitting or pre-check approach would require
- [general] `bitcask-crash-recovery-on-rotation` — What happens if the process crashes between `close()` and `_open_active_file()`? The ID increment is in-memory only
- [file] `hash-index-storage/test_bitcask.py` — Tests that exercise rotation behavior and verify correctness across file boundaries

## Beliefs

- `bitcask-single-active-writer` — At most one data file is open for writes at any time; rotation closes the old before opening the new
- `bitcask-rotation-is-soft-cap` — File rotation is checked before each write, so the active file can exceed `max_file_size` by up to one record
- `bitcask-rotation-preserves-read-handles` — When the active file is rotated, its read handle in `file_handles` is preserved so existing `KeyEntry` references remain valid
- `bitcask-sequential-file-ids` — File IDs are assigned by incrementing `active_file_id` by 1 with no gap detection or collision check

