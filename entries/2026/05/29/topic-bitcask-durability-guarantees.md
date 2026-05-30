# Topic: The gap between `flush()` and `fsync()` and what it means for crash recovery in this implementation

**Date:** 2026-05-29
**Time:** 10:01

# The Gap Between `flush()` and `fsync()` in Crash Recovery

## The Core Distinction

`flush()` and `fsync()` operate at different layers of the I/O stack, and confusing them is a classic durability bug:

- **`file.flush()`** pushes data from Python's userspace buffer into the **OS kernel's page cache**. After this call, other processes can read the data, but it may still be sitting in RAM — not yet on disk.
- **`os.fsync(fd)`** forces the kernel to write dirty pages for that file descriptor to **persistent storage**. Only after `fsync()` returns can you trust that data will survive a power loss or kernel crash.

The window between `flush()` and `fsync()` is the durability gap. Data in the kernel page cache looks committed — reads return it, `stat()` shows the new file size — but a crash erases it.

## How Each Module Handles the Gap

### The WAL module: Gets it right

`write-ahead-log/wal.py` consistently pairs `flush()` with `fsync()`. The `_do_sync` method at lines 122–133 shows the deliberate pattern:

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

Every critical operation — `append` (line 147), `append_batch` (line 163), `checkpoint` (line 170), `truncate` (line 183), `_rotate` (lines 114–115) — goes through `_do_sync`. The `batch` sync mode is the only intentional durability tradeoff: it accumulates up to `batch_sync_count` writes before fsyncing, trading some crash exposure for throughput.

### The B-tree's PageManager: flush-only writes, selective fsync

`b-tree-storage-engine/btree.py` reveals a split personality. The `PageManager` calls `flush()` after every page write (lines 46, 54, 80) but only calls `fsync()` in two explicit methods:

- **`sync()`** at lines 104–105: `self._f.flush()` then `os.fsync(self._f.fileno())`
- **`close()`** at lines 112–113: same pattern

Regular page writes via `write_page` (line 80), `_write_meta` (line 46), and `_write_empty_leaf` (line 54) call only `flush()`. This means that page data lives in the kernel page cache without a durability guarantee until someone explicitly calls `sync()` or `close()`.

The B-tree's **WAL class** (the inner `WAL` at line 117, distinct from the standalone WAL module) compensates. Its `log_write` method at lines 136–137 does `flush()` + `fsync()` on every WAL entry. The protocol is:

1. Write the page image to the WAL, `fsync()` the WAL (line 136–137)
2. Write the actual page to the data file, `flush()` only (line 80)
3. On `commit()` (line 140), `sync()` the data file, **then** truncate the WAL

If the system crashes between steps 2 and 3, the WAL `recover()` method (line 148) replays the logged page images into the data file and fsyncs (line 170–171). This is the classic WAL protocol — the data file doesn't need fsync on every write because the WAL is the durability backstop.

### The LSM tree's WAL: The durability hole

`log-structured-merge-tree/lsm.py` has a significant gap. Its `WAL.append` method at line 26:

```python
def append(self, key: str, value: bytes):
    ...
    self._fd.flush()
```

**No `fsync()`.** The entire LSM WAL never calls `os.fsync()`. This means the WAL — the very component responsible for crash recovery — only pushes data to the kernel page cache. A power failure can lose WAL entries that `flush()` appeared to commit.

The `test_crash_recovery` test at `log-structured-merge-tree/test_lsm.py:92` simulates a crash by reopening the LSMTree without calling `close()`. This test passes because it runs in-process — the kernel page cache still holds the flushed data. A real power failure would expose the gap.

### Bitcask: Configurable durability

`hash-index-storage/bitcask.py` at lines 86–88 takes the explicit approach:

```python
self.active_file.flush()
if self.sync_writes:
    os.fsync(self.active_file.fileno())
```

The `sync_writes` flag (default `True`) lets callers choose their durability/performance tradeoff. With `sync_writes=False`, Bitcask has the same gap as the LSM WAL. With `True`, every record is durable before `put()` returns.

## What This Means for Crash Recovery

The implementations fall into three categories:

| Module | Strategy | Crash-safe? |
|--------|----------|-------------|
| `write-ahead-log/wal.py` | `flush()` + `fsync()` on every sync point | Yes (with batch mode tradeoff) |
| `b-tree-storage-engine/btree.py` | WAL fsyncs; data file flush-only until commit | Yes — WAL replay covers the gap |
| `hash-index-storage/bitcask.py` | Configurable via `sync_writes` | Depends on configuration |
| `log-structured-merge-tree/lsm.py` | `flush()` only, no `fsync()` anywhere | **No** — WAL entries can be lost |

The LSM tree's missing `fsync()` is the most consequential finding. The WAL is the foundation of its crash recovery story — `replay()` at line 28 rebuilds the memtable from WAL entries on startup. If those entries never made it to disk, replay returns an incomplete dataset and the user sees silent data loss.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_do_sync` — The three sync modes (sync, batch, none) and how batch mode trades durability for throughput
- [function] `b-tree-storage-engine/btree.py:recover` — How WAL replay reconstructs the data file after a crash, and the ordering invariant between WAL fsync and data file commit
- [file] `log-structured-merge-tree/test_lsm.py` — The crash recovery test that passes despite the missing fsync — understanding why in-process "crash" tests can mask real durability bugs
- [general] `directory-fsync-gap` — None of the implementations fsync the parent directory after creating new files (WAL rotation, SSTable creation), which means new file entries may not survive a crash on ext4/XFS
- [function] `hash-index-storage/bitcask.py:compact` — Whether compaction output files are fsynced before the old files are deleted — a race where losing both old and new data is possible

## Beliefs

- `lsm-wal-missing-fsync` — The LSM tree WAL (`log-structured-merge-tree/lsm.py:26`) calls `flush()` but never `fsync()`, so WAL entries are not durable against power failure
- `btree-wal-fsync-before-data-write` — The B-tree WAL fsyncs each logged page image (`btree.py:136-137`) before the corresponding data file page write, ensuring replay can recover from crashes during page writes
- `standalone-wal-always-pairs-flush-fsync` — The standalone WAL module (`write-ahead-log/wal.py`) pairs every `flush()` with an `os.fsync()` call in its `_do_sync` method, with the sole exception of batch mode deferral
- `bitcask-fsync-is-configurable` — Bitcask's `sync_writes` constructor parameter (default `True`) controls whether `os.fsync()` follows each `flush()`, making durability a caller's choice
- `btree-page-writes-not-individually-fsynced` — B-tree `PageManager.write_page` calls only `flush()`, relying on the WAL for crash safety rather than fsyncing each page write to the data file

