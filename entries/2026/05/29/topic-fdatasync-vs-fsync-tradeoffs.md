# Topic: `os.fdatasync()` skips inode metadata (mtime/size) and is faster; explore where the codebase could use it instead of `os.fsync()` without sacrificing correctness

**Date:** 2026-05-29
**Time:** 06:38

# `os.fdatasync()` vs `os.fsync()` in the DDIA Implementations

## The Core Difference

`os.fsync(fd)` flushes **both file data and all inode metadata** (mtime, ctime, size, permissions) to stable storage. `os.fdatasync(fd)` flushes **file data plus only the metadata needed to retrieve that data** — specifically, it updates size if the file grew, but skips mtime/ctime/permissions. On Linux, when you overwrite existing bytes without extending the file, `fdatasync` avoids the metadata write entirely — that's where the real speedup lives.

## The 13 `fsync` Call Sites

The codebase has 13 `os.fsync()` calls and zero `os.fdatasync()` calls. Here's the breakdown by safety of replacement:

### Safe to replace — and a meaningful win

**B-tree in-place page overwrites** (`b-tree-storage-engine/btree.py`):

- **Line 105** (`PageManager.sync()`): Called after writing pages. When the B-tree updates or splits existing pages, it overwrites at fixed offsets without growing the file. The file size doesn't change, so `fdatasync` skips the metadata I/O entirely. This is the biggest performance opportunity in the codebase — B-trees do frequent in-place page writes during updates, and each one currently pays for an unnecessary metadata flush.
- **Line 113** (`PageManager.close()`): Final sync before close. Same analysis — if the last operation was an in-place rewrite, `fdatasync` wins.

### Safe to replace — modest win

These are all **append-only writes** where the file size changes. `fdatasync` still flushes the size update (it must, to read the data back correctly), but it skips mtime/ctime — a small saving per call:

| File | Line | Context |
|------|------|---------|
| `write-ahead-log/wal.py` | 115 | `_rotate()` — flush before closing old WAL segment |
| `write-ahead-log/wal.py` | 128 | `_do_sync()` — per-record sync in `"sync"` mode |
| `write-ahead-log/wal.py` | 133 | `_do_sync()` — batch sync when count threshold met |
| `write-ahead-log/wal.py` | 184 | `truncate()` — flush before closing for rewrite |
| `write-ahead-log/wal.py` | 209 | `truncate()` — sync the rewritten file |
| `write-ahead-log/wal.py` | 261 | `close()` — final flush |
| `b-tree-storage-engine/btree.py` | 137 | `WAL.log_write()` — append WAL entry |
| `hash-index-storage/bitcask.py` | 88 | `_write_record()` — append to active data file |

### Safe but needs care

- **`btree.py` line 144** (`WAL.commit()`): After `truncate(0)`, the file shrinks to zero bytes. `fdatasync` will flush this size change, so it's safe — the truncation is durable. But this is the one case where you might want to keep `fsync` for extra caution, since a lost truncation means replaying stale WAL entries (idempotent but slow).
- **`btree.py` line 171** (`WAL.recover()`): Same pattern — truncate-then-sync after replay. Same reasoning applies.

## Where the Win Is Largest

The **B-tree `PageManager`** is the standout candidate. Here's why:

1. B-trees do **in-place overwrites** of fixed-size pages — the file size doesn't change for most writes
2. `PageManager.sync()` at line 105 is called on every committed write path
3. Skipping the metadata write means one fewer disk seek per sync on rotational media, and one fewer journal entry on ext4

For the WAL and Bitcask modules, the savings are real but smaller — they append, so the size metadata must be flushed regardless.

## A Notable Gap: LSM Tree WAL

The LSM tree's WAL (`log-structured-merge-tree/lsm.py`, line 26) calls `self._fd.flush()` but **never calls `fsync` or `fdatasync`**. This means its WAL entries can survive in the OS page cache without reaching stable storage — a durability gap. This isn't an `fdatasync` candidate; it's a missing sync entirely.

## Platform Caveat

`os.fdatasync()` is a POSIX call available on Linux and macOS. On **macOS**, however, both `fsync` and `fdatasync` have the same weak semantics (neither guarantees write-through to the physical drive platter; you need `fcntl(fd, F_FULLFSYNC)` for that). So the `fdatasync` optimization only provides a real benefit on **Linux with ext4/xfs/btrfs**. A production implementation would want:

```python
import sys

def _sync_data(fd):
    if sys.platform == "linux":
        os.fdatasync(fd.fileno())
    else:
        os.fsync(fd.fileno())
```

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:PageManager.sync` — The highest-impact replacement site; trace all callers to confirm no path depends on mtime being flushed
- [function] `log-structured-merge-tree/lsm.py:WAL.append` — Missing fsync entirely; understand whether the LSM tree's durability model intentionally relies on memtable-level recovery only
- [general] `directory-fsync-after-rename` — None of the rotation/compaction code fsyncs the parent directory after creating new files; on Linux, the new directory entry can be lost on crash even if the file data is synced
- [function] `write-ahead-log/wal.py:_do_sync` — The `sync_mode` parameter controls whether sync happens per-record or in batches; understand how batch mode interacts with fdatasync to quantify the combined throughput improvement
- [file] `hash-index-storage/bitcask.py` — The `sync_writes` flag already makes fsync optional; explore whether fdatasync should be the default when `sync_writes=True` rather than full fsync

## Beliefs

- `btree-page-overwrite-no-size-change` — B-tree PageManager overwrites pages at fixed offsets within a pre-allocated file, so most write+sync cycles don't change file size and would benefit from fdatasync skipping metadata I/O
- `lsm-wal-missing-fsync` — The LSM tree WAL (`lsm.py:WAL.append`, line 26) calls `flush()` without any fsync/fdatasync, meaning WAL entries are not guaranteed durable on crash
- `all-fsync-sites-data-integrity-only` — None of the 13 fsync call sites have callers that depend on mtime or ctime metadata being accurate; all syncs exist purely for data durability, making every site a valid fdatasync candidate
- `no-directory-fsync-after-file-creation` — WAL rotation (`wal.py:_rotate`) and Bitcask compaction create new files but never fsync the parent directory, so the new filename can be lost on Linux crash even with per-file fsync

