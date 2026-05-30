# Topic: The `_do_sync` method (`wal.py:131–140`) has three sync modes (sync/batch/none) with different durability tradeoffs; `append_batch` and `checkpoint` both force sync regardless of mode, which is worth understanding

**Date:** 2026-05-29
**Time:** 11:27

I'll work with what was captured in the observations, noting the gaps.

---

## The `_do_sync` Method: Durability vs. Performance

The WAL's `_do_sync` method (around `wal.py:125–133`) is where the system decides *when data actually hits disk*. This is the core durability knob.

### The three sync modes

```python
if self._sync_mode == "sync" or force:
    self._fd.flush()
    os.fsync(self._fd.fileno())
elif self._sync_mode == "batch":
    self._write_count += 1
    if self._write_count >= self._batch_sync_count:
        self._fd.flush()
        os.fsync(self._fd.fileno())
        self._write_count = 0
```

| Mode | Behavior | Durability | Performance |
|------|----------|------------|-------------|
| `"sync"` | `flush()` + `fsync()` on every call | Strongest — every record is on disk before returning | Slowest — one syscall pair per write |
| `"batch"` | `flush()` + `fsync()` every N writes (`_batch_sync_count`) | Window of up to N-1 records can be lost on crash | Middle ground — amortizes fsync cost |
| `"none"` (implicit else) | No flush, no fsync | Weakest — data sits in OS page cache, lost on power failure | Fastest — pure userspace writes |

### Why `flush()` *and* `fsync()`?

These are two distinct operations that people often confuse:

- **`flush()`** pushes data from Python's internal buffer into the OS kernel buffer. Without this, data may not even reach the OS.
- **`os.fsync()`** tells the OS to write kernel buffers to the physical storage device. Without this, a power loss can lose data that the OS was holding in page cache.

You need both. `flush()` alone doesn't guarantee durability. `fsync()` alone won't help if Python is still holding data in its own buffer.

### The `force` parameter

Notice the first condition: `self._sync_mode == "sync" or force`. The `force` flag bypasses the mode entirely and always syncs. This is the mechanism that `append_batch` and `checkpoint` would use — they pass `force=True` to `_do_sync`, ensuring the data is on disk regardless of what sync mode the WAL was configured with.

This makes sense: a batch commit or a checkpoint is a durability boundary. If you're checkpointing, you're saying "everything up to here is safe." That guarantee is meaningless if the checkpoint record itself could be lost from a page cache.

### How `append` uses it

In the `append` method (`wal.py:141–150`), every single record write calls `_do_sync()` *without* `force`:

```python
self._fd.write(data)
self._do_sync()
self._maybe_rotate()
```

So individual appends respect the configured sync mode. In `"none"` mode, `append` just writes and moves on — the data might not be on disk for a while. In `"sync"` mode, every append is a full fsync round-trip.

### What's missing from the observations

The observations didn't capture the `append_batch` or `checkpoint` method bodies (both returned 0 definitions). Based on the pattern, these methods almost certainly call `self._do_sync(force=True)`. It would be worth reading the full file to confirm this and to see whether `append_batch` wraps multiple records in a single fsync (which would be the whole point — batch the writes, sync once).

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:append_batch` — Confirm it calls `_do_sync(force=True)` and understand how it batches multiple records into a single fsync for atomicity
- [function] `write-ahead-log/wal.py:checkpoint` — Understand what a WAL checkpoint does (likely writes a marker record and forces sync, enabling truncation of older segments)
- [function] `write-ahead-log/wal.py:_maybe_rotate` — File rotation at `_max_file_size` interacts with sync: does the old file get an fsync before rotation closes it?
- [general] `wal-crash-recovery` — How does the WAL reader handle a crash that left a partial record at the end of a file? The CRC check in `_read_record` is part of this story
- [file] `write-ahead-log/wal.py` — Read the `__init__` constructor to see default values for `_sync_mode`, `_batch_sync_count`, and `_max_file_size`

---

## Beliefs

- `wal-sync-mode-flush-plus-fsync` — In `"sync"` mode, `_do_sync` always calls both `fd.flush()` and `os.fsync()` — flush alone does not guarantee durability since data may remain in OS page cache
- `wal-batch-sync-counter-reset` — In `"batch"` mode, `_do_sync` resets `_write_count` to 0 after each fsync, meaning up to `_batch_sync_count - 1` records can be lost on crash
- `wal-none-mode-no-sync` — When `_sync_mode` is neither `"sync"` nor `"batch"`, `_do_sync` performs no flush or fsync, leaving durability entirely to the OS page cache flush schedule
- `wal-force-sync-overrides-mode` — The `force` parameter in `_do_sync` bypasses the configured sync mode, ensuring flush+fsync happens regardless — this is the mechanism for durability boundaries like commits and checkpoints
- `wal-append-respects-sync-mode` — Individual `append` calls invoke `_do_sync()` without `force=True`, so their durability depends entirely on the configured sync mode

