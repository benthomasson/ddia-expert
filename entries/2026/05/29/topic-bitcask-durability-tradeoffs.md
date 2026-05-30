# Topic: The relationship between `sync_writes`, `flush()`, and `fsync()` — and what guarantees each level actually provides on different filesystems

**Date:** 2026-05-29
**Time:** 11:59

# `sync_writes`, `flush()`, and `fsync()`: Three Levels of Durability

## The Core Distinction

There are three levels of "making sure data is written," and this codebase uses all three — sometimes correctly, sometimes not:

| Level | Call | What it does | Survives process crash | Survives power loss |
|-------|------|-------------|----------------------|-------------------|
| 1 | `write()` only | Puts data in Python's userspace buffer | No | No |
| 2 | `flush()` | Pushes Python buffer → OS kernel page cache | Yes | **No** |
| 3 | `flush()` + `os.fsync()` | Pushes kernel page cache → physical disk | Yes | **Yes** |

The critical insight: **`flush()` alone does NOT guarantee durability.** It only moves data from your process into the kernel. If the machine loses power, the kernel's page cache is gone. Only `fsync()` forces the OS to write through to stable storage.

## How Each Module Uses These Levels

### Bitcask — The Cleanest Pattern

`hash-index-storage/bitcask.py:82-88` shows the canonical two-tier approach:

```python
self.active_file.flush()          # always: survive process crash
if self.sync_writes:
    os.fsync(self.active_file.fileno())  # optional: survive power loss
```

`flush()` is **unconditional** (line 86). `fsync()` is gated behind the `sync_writes` flag (line 87-88, default `True`). This gives callers a performance knob: tests pass `sync_writes=False` (every single test does this) because fsync is slow and tests don't need power-loss durability. Production code leaves it at `True`.

### WAL — The Most Sophisticated

`write-ahead-log/wal.py:125-133` implements three sync modes:

```python
def _do_sync(self, force: bool = False):
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

- **`"sync"` mode**: flush+fsync on every write. Maximum durability, worst throughput.
- **`"batch"` mode**: flush+fsync every N writes (default 100). You can lose up to N-1 records on power loss, but throughput improves dramatically.
- **`force=True`**: overrides batch mode for critical writes. Used by `append_batch` (line 183-184) and `checkpoint` (line 208-209) — operations where partial loss would corrupt semantics.

Notice that `_rotate` (line 114-115) always does flush+fsync before closing the old file, regardless of mode. This is correct — you must ensure the old file is fully durable before writing to a new one, or a crash mid-rotation could lose data from both.

### B-tree PageManager — flush Without fsync

`b-tree-storage-engine/btree.py:46,54,80` — the `PageManager` calls `flush()` after every page write but does **not** call `fsync()`. The explicit `sync()` method (lines 104-105) is a separate call:

```python
def sync(self):
    self._f.flush()
    os.fsync(self._f.fileno())
```

This is intentional: individual page writes during a B-tree operation go to the kernel (visible to a re-read) but aren't forced to disk until `sync()` is called after the WAL commits or at `close()` (lines 112-113, 143-144). The WAL itself (inner class, lines 136-137) does flush+fsync on every `log_write` — because the WAL is the durability mechanism, and the data file can be reconstructed from it.

### LSM WAL — The Dangerous One

`log-structured-merge-tree/lsm.py:26` — this WAL only calls `flush()`:

```python
def append(self, key: str, value: bytes):
    ...
    self._fd.flush()
```

No `fsync()` anywhere in the LSM WAL. This means the WAL data reaches the kernel page cache but is **not guaranteed to survive a power failure**. For a write-ahead log — whose entire purpose is crash recovery — this is a durability gap. A process crash is fine (the kernel still has the data), but a kernel panic or power loss could silently lose WAL entries that the application thought were committed.

### Event Store — No Durability Guarantees At All

`event-sourcing-store/event_store.py:130-137` — `_persist_event` opens, writes, and closes without any explicit flush or fsync. It relies on Python's context manager (`with open(...)`) to close the file, which calls `close()` → implicit `flush()`. No `fsync()`. Even the implicit flush only happens at the `with` block boundary. For an event store, this means events could be lost on either process crash or power loss.

## Filesystem Caveats

Even `fsync()` doesn't mean the same thing everywhere:

- **ext4 (default Linux)**: `fsync()` on the file does NOT guarantee the directory entry is durable. If you create a new file, you need to `fsync()` the parent directory too. The WAL rotation code (`wal.py:114-115`) does fsync the old file but does NOT fsync the directory after creating the new one.
- **macOS (HFS+/APFS)**: `fsync()` historically only flushes to the drive's write cache, not to platters. `fcntl(F_FULLFSYNC)` is needed for true durability. None of these implementations use it.
- **ZFS/btrfs (CoW filesystems)**: fsync semantics are generally stronger — a fsync implicitly ensures metadata consistency because writes are copy-on-write.
- **NFS**: `fsync()` behavior depends on the mount options and server implementation. It may or may not guarantee anything.

None of these implementations address directory fsync or platform-specific flush semantics, which is acceptable for reference implementations but would be a concern in production.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_do_sync` — Compare the three sync modes and trace which callers use `force=True` vs relying on the mode
- [function] `log-structured-merge-tree/lsm.py:WAL.append` — Investigate whether the missing fsync is intentional or a bug, and what recovery guarantees the LSM tree actually provides
- [function] `b-tree-storage-engine/btree.py:WAL.commit` — Understand the WAL commit protocol: why does it sync the page manager, then truncate, then fsync the truncation?
- [general] `directory-fsync-gap` — None of the rotation/file-creation paths fsync the parent directory, which on ext4 means new files may not survive power loss even if their contents are fsynced
- [file] `event-sourcing-store/event_store.py` — Compare its persistence approach against bitcask and WAL to understand why an event store might accept weaker durability guarantees

## Beliefs

- `bitcask-flush-always-fsync-conditional` — Bitcask unconditionally calls `flush()` on every write but only calls `os.fsync()` when `sync_writes=True`, making `flush()` the baseline durability and `fsync()` an opt-in for power-loss safety
- `wal-batch-sync-force-override` — The WAL's batch sync mode is overridden by `force=True` on `append_batch` and `checkpoint`, ensuring atomic batch boundaries are always fsynced regardless of the configured sync mode
- `lsm-wal-missing-fsync` — The LSM tree's WAL (`lsm.py:26`) calls `flush()` but never `fsync()`, meaning it cannot guarantee crash recovery after kernel panic or power loss — a weaker guarantee than the standalone WAL module
- `btree-wal-syncs-data-file-defers` — The B-tree PageManager calls `flush()` on every page write but defers `fsync()` to explicit `sync()` calls, while its inner WAL class fsync's every log entry — the WAL is the durability mechanism, the data file is reconstructable
- `no-directory-fsync-anywhere` — No module in the codebase fsync's the parent directory after creating new files (during rotation or SSTable creation), which on ext4 means the directory entry itself may not survive a power failure

