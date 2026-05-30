# Topic: The correctness of this WAL depends on `fsync` ordering between the WAL file and data file; explore what guarantees the OS and filesystem actually provide

**Date:** 2026-05-29
**Time:** 10:29

# fsync Ordering Between WAL and Data Files

## The Core Contract

A write-ahead log's correctness rests on one invariant: **the WAL record must be durable on disk before the corresponding data file mutation becomes durable**. If the data file change lands first and the system crashes, the WAL can't replay the operation — the data file is in a new state but the WAL doesn't know what happened to get there. If the WAL record lands first and the system crashes, recovery replays the WAL and re-applies the mutation. The order of `fsync` calls is what enforces this.

## What This Code Actually Does

### The standalone WAL (`write-ahead-log/wal.py`)

The `_do_sync` method at **line 127-134** is the sync decision point:

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

In `sync` mode, every `append()` call (line ~145) does `flush()` + `os.fsync()`. This is correct for the WAL file itself. But this WAL is a standalone module — it doesn't coordinate with any data file. The caller is responsible for the ordering: write WAL, fsync WAL, mutate data, fsync data. **Nothing in this code enforces or documents that protocol.**

### The LSM tree's WAL (`log-structured-merge-tree/lsm.py`)

This is the critical problem. The WAL at **line 26** does:

```python
self._fd.flush()
```

**No `os.fsync()` call.** The `flush()` only pushes data from Python's userspace buffer into the kernel's page cache. The kernel is free to reorder or delay the actual disk write indefinitely. On a power failure, every WAL record written since the last kernel writeback is lost. This WAL provides no crash durability at all — it protects only against Python process crashes (where the kernel survives and eventually writes back its pages).

### Bitcask (`hash-index-storage/bitcask.py`)

Bitcask at **line 95-96** does:

```python
self.active_file.flush()
if self.sync_writes:
    os.fsync(self.active_file.fileno())
```

Bitcask sidesteps the WAL-vs-data ordering problem entirely: the append-only data file *is* the log. There's only one file to fsync. This is architecturally clean but means Bitcask doesn't need to solve the harder problem.

## What the OS and Filesystem Actually Guarantee

### `fsync` guarantees (POSIX)

`os.fsync(fd)` asks the kernel to flush all modified data **and metadata** for that file descriptor to stable storage. POSIX says that when `fsync` returns, the data is durable. But there are important caveats:

1. **`fsync` is per-file, not cross-file.** Calling `fsync` on the WAL file guarantees the WAL is durable. It says nothing about the data file. To get ordering, you must: `fsync(wal_fd)` → write data → `fsync(data_fd)`. The two fsyncs create the ordering.

2. **`fsync` doesn't sync the directory entry.** When you create a new file (as `_rotate` does at **wal.py line 114-120**), the file's contents may be durable but the *directory entry pointing to it* may not be. On ext4, if you fsync the new WAL file but not the directory, a crash can result in the file's contents existing on disk with no directory entry — the file is unreachable. The grep for `dir.*fsync|fsync.*dir` returned **zero matches** across the entire codebase. Every `_rotate()` call, every truncation rewrite, and every SSTable creation is vulnerable to this.

3. **`fdatasync` vs `fsync`.** `fdatasync` syncs data but skips metadata (timestamps, permissions) unless the file size changed. For append workloads where the file grows, `fdatasync` still has to sync the size change — so the saving is minimal. For in-place overwrites it's faster. This codebase never uses `fdatasync` (zero matches in grep). For a WAL that's always appending, `fsync` is the correct choice anyway.

### Filesystem-specific behaviors

- **ext4 with `data=ordered` (the default):** Data blocks are written before the metadata transaction commits. This provides some implicit ordering — you won't see a file with stale data blocks — but it does *not* replace `fsync`. A crash can still lose recently-written data that was in the page cache.

- **ext4 with `data=writeback`:** No ordering between data and metadata at all. A crash can expose stale or garbage data blocks. This makes the missing fsyncs in the LSM WAL even more dangerous.

- **XFS:** Historically aggressive about delayed allocation. Famously had zero-length file bugs on crash before barriers were required. Modern XFS is better, but directory fsync is still required for new file visibility.

- **macOS (APFS/HFS+):** `fsync` on macOS traditionally did *not* flush the disk's write cache — only `fcntl(fd, F_FULLFSYNC)` did. As of recent macOS versions, `fsync` behavior has improved, but `F_FULLFSYNC` remains the only guaranteed-durable call. **This codebase runs on macOS (Darwin 24.4.0) and uses `os.fsync()` everywhere, never `F_FULLFSYNC`.** On macOS, none of the fsync calls may actually be durable against power loss.

## The Missing Pieces

| Gap | Location | Impact |
|-----|----------|--------|
| LSM WAL never fsyncs | `lsm.py:26` | WAL provides zero crash durability |
| No directory fsync after rotation | `wal.py:114-120` | New WAL file can vanish on crash |
| No directory fsync after truncation | `wal.py:183-209` (truncate rewrites files) | Rewritten files can vanish |
| No `F_FULLFSYNC` on macOS | All `os.fsync()` calls | Not truly durable on macOS |
| No cross-file ordering protocol | Entire codebase | Callers must implement WAL→data ordering themselves, undocumented |

## The `batch` and `none` Sync Modes

The `batch` mode at **wal.py line 131-134** accumulates writes and only fsyncs every `batch_sync_count` operations. Records between syncs are vulnerable. This is a legitimate performance tradeoff (group commit), but the window of vulnerability should be documented. The `none` mode never fsyncs at all — the WAL is purely a logical log for process crash recovery, not power failure recovery.

## Summary

The standalone WAL in `wal.py` does the right thing for its own file in `sync` mode — but it doesn't solve the harder problem of cross-file ordering, and it's missing directory fsyncs for file creation. The LSM WAL in `lsm.py` doesn't fsync at all, making it ineffective against anything beyond a process crash. And on macOS, `os.fsync()` may not actually reach stable storage without `F_FULLFSYNC`.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:WAL.append` — The WAL that never fsyncs; trace how this affects LSM crash recovery end-to-end
- [general] `directory-fsync-gap` — What happens to newly rotated WAL files and SSTables when the directory isn't fsynced
- [function] `write-ahead-log/wal.py:WriteAheadLog.truncate` — Rewrites files in place during truncation; explore the atomicity gap if a crash occurs mid-rewrite
- [general] `macos-fsync-semantics` — How `F_FULLFSYNC` differs from `fsync` on Darwin/APFS and whether this codebase needs it
- [file] `hash-index-storage/bitcask.py` — Compare the append-only-file-as-its-own-WAL approach against the dual-file WAL+data pattern

## Beliefs

- `lsm-wal-no-fsync` — The LSM tree's WAL (`lsm.py:26`) calls `flush()` but never `os.fsync()`, providing no durability against kernel crashes or power loss
- `no-directory-fsync-anywhere` — No module in the codebase fsyncs directory file descriptors after creating new files, leaving new WAL segments and SSTables vulnerable to disappearing on crash
- `wal-sync-mode-batch-window` — In batch sync mode (`wal.py:131-134`), up to `batch_sync_count - 1` records can be lost on crash because fsync is deferred
- `no-cross-file-fsync-ordering` — The WAL module provides no mechanism to enforce the WAL-before-data fsync ordering; callers must implement this protocol themselves with no documented guidance
- `macos-fsync-not-durable` — On macOS (the current platform), `os.fsync()` may not flush the disk write cache; true durability requires `fcntl(fd, F_FULLFSYNC)` which is never used

