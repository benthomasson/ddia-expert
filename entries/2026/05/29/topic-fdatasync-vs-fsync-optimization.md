# Topic: For append-only files like WALs, `fdatasync` skips unnecessary metadata updates (mtime) and can be 2-3x faster; explore whether switching would be safe here

**Date:** 2026-05-29
**Time:** 10:04

# `fdatasync` vs `fsync` in Append-Only Storage Files

## What's the difference?

`fsync` flushes both file **data** and all **metadata** (mtime, atime, permissions, size) to stable storage. `fdatasync` flushes file data and only the metadata required for subsequent data retrieval — in practice, file size if it changed, but **not** mtime. For append-only workloads where every write grows the file, `fdatasync` does everything you need while skipping an extra inode write for timestamps. On Linux ext4, this can eliminate a second disk barrier per sync, yielding 2-3x throughput improvement under heavy write loads.

## Where `fsync` is used today

The codebase has three distinct sync profiles:

### 1. Heavy sync — `write-ahead-log/wal.py`

The standalone WAL uses `os.fsync()` on nearly every write path. The `_do_sync` method at line 126-133 is the central dispatch:

```python
# wal.py:126-133
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

Additional `os.fsync()` calls appear in `_rotate` (line 115), `truncate` (line 184), `checkpoint` via `_do_sync` (line 183-184), and the truncation rewrite loop (line 208-209). This file is **append-only** — every write appends records, and rotation creates a new file. **Strong candidate for `fdatasync`.**

### 2. Append-only data files — `hash-index-storage/bitcask.py`

The `_write_record` method (line 85-93) does:
```python
self.active_file.flush()
if self.sync_writes:
    os.fsync(self.active_file.fileno())
```

Bitcask data files are strictly append-only. The file is opened with `"ab"` mode (line 68). **Safe for `fdatasync`.**

### 3. B-tree WAL vs page file — `b-tree-storage-engine/btree.py`

The B-tree has two sync paths:
- **WAL** (`log_write` at line 138): append-only — `fdatasync` safe
- **PageManager** (`sync` at line 112, `close` at line 117): does **in-place page writes** (`write_page` seeks to specific offsets). However, `fdatasync` still flushes data for in-place writes — it only skips non-essential metadata. **Also safe for `fdatasync`**, since the page file's size doesn't change during normal writes (pages are pre-allocated).

### 4. Missing sync entirely — `log-structured-merge-tree/lsm.py`

The LSM WAL class (line 13-63) only calls `self._fd.flush()` at line 26 — **no `os.fsync()` or `os.fdatasync()` at all**. This is a durability gap: `flush()` only moves data from Python's userspace buffer to the OS page cache; a crash could still lose data. Before considering `fdatasync`, this file needs *any* sync call.

### 5. No sync — `log-structured-hash-table/bitcask.py` and `event-sourcing-store/event_store.py`

`log-structured-hash-table/bitcask.py` calls `flush()` at line 155 but no `fsync`. The event store (line 123) opens with `"a"` and writes JSON lines without any sync. These have the same durability gap as the LSM WAL.

## Is switching safe?

**Yes, for all append-only paths**, with two caveats:

1. **Platform availability**: Python's `os.fdatasync()` exists on Linux but **not on macOS/Darwin** (the current platform). A portable implementation needs a fallback:

   ```python
   _fdatasync = getattr(os, 'fdatasync', os.fsync)
   ```

2. **File creation requires directory sync**: When creating new WAL segments (`_rotate` in `wal.py:112-122`) or new SSTable files, `fdatasync` on the file alone doesn't guarantee the **directory entry** is durable. A crash between file creation and the next directory sync could leave an orphaned inode. The current code doesn't `fsync` directories either, so this is a pre-existing gap — but switching to `fdatasync` doesn't make it worse.

3. **Truncation paths need `fsync`**: The WAL truncation in `wal.py:190-219` rewrites files entirely and may change file size downward. `fdatasync` handles size changes, so it's still technically safe, but truncation is rare enough that the performance benefit is negligible — keeping `fsync` for truncation is the conservative choice.

## Recommendation

Replace `os.fsync` with `os.fdatasync` (with a fallback) in the append and sync paths of:
- `write-ahead-log/wal.py:_do_sync` — highest impact, called on every write in sync mode
- `hash-index-storage/bitcask.py:_write_record` — called on every put/delete
- `b-tree-storage-engine/btree.py:WAL.log_write` — called on every B-tree mutation

Leave `os.fsync` in place for:
- WAL truncation/rewrite paths
- `PageManager.close()` (final shutdown sync)
- Any path that creates a new file (and add directory fsync there too)

Fix the **missing sync** in `log-structured-merge-tree/lsm.py:WAL.append` and `log-structured-hash-table/bitcask.py:_write_record` — these should call at least `fdatasync` after `flush()`.

---

## Topics to Explore

- [general] `directory-fsync-gap` — None of these storage engines fsync the parent directory after creating new files (WAL segments, SSTables); explore whether this leaves a crash window where a newly-rotated file vanishes
- [function] `log-structured-merge-tree/lsm.py:WAL.append` — This WAL calls `flush()` but never `fsync`/`fdatasync`, meaning data can be lost on crash despite having a WAL — a correctness bug worth investigating
- [function] `write-ahead-log/wal.py:_do_sync` — Central sync dispatch with three modes (sync/batch/none); the right place to implement an `fdatasync` switch and benchmark the difference
- [file] `b-tree-storage-engine/btree.py` — The B-tree mixes append-only WAL writes with in-place page writes in the same module; understanding which paths are truly append-only matters for choosing the right sync primitive
- [general] `darwin-fsync-semantics` — On macOS, `fsync` doesn't guarantee disk write cache flush (need `fcntl F_FULLFSYNC`); the durability guarantees of this entire codebase may be weaker than intended on the current platform

## Beliefs

- `wal-py-fsync-on-every-sync-write` — `write-ahead-log/wal.py:_do_sync` calls `os.fsync` after every record in `sync` mode and every N records in `batch` mode; switching these to `fdatasync` would be the highest-impact optimization
- `lsm-wal-missing-fsync` — `log-structured-merge-tree/lsm.py:WAL.append` (line 26) calls only `flush()` with no `fsync` or `fdatasync`, creating a durability gap where acknowledged writes can be lost on crash
- `fdatasync-safe-for-all-append-paths` — Every file opened in append mode (`"ab"`) in this codebase writes sequentially with monotonically increasing file size, making `fdatasync` a safe drop-in replacement for `fsync` on those paths
- `fdatasync-not-available-on-darwin` — Python's `os.fdatasync` is only available on Linux; any switch requires a platform fallback (`getattr(os, 'fdatasync', os.fsync)`) to work on the current macOS development environment
- `no-directory-fsync-anywhere` — No module in the codebase calls `fsync` on a directory file descriptor after creating new files (WAL rotation, SSTable creation, segment rotation), leaving all file creation operations vulnerable to a crash-induced metadata loss

