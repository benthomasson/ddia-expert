# Topic: Neither buggy nor correct implementations fsync the *parent directory* after creating new segment/WAL files — a subtler durability gap where the file's directory entry can be lost

**Date:** 2026-05-29
**Time:** 10:49

# The Directory Fsync Gap

## The Problem

On POSIX filesystems, creating a new file involves two distinct metadata changes: the file's own inode (size, timestamps, data block pointers) and the **parent directory's entry** that maps the filename to that inode. Calling `os.fsync(fd)` on a file descriptor flushes the file's data and metadata — but *not* the directory entry. To make the filename durable, you must also open the parent directory and fsync *that* file descriptor:

```python
dir_fd = os.open(parent_dir, os.O_RDONLY)
os.fsync(dir_fd)
os.close(dir_fd)
```

None of the implementations in this codebase do this.

## Where It Matters

### WAL Rotation (`write-ahead-log/wal.py:111-122`)

The `_rotate` method creates a new `.wal` file each time the current file exceeds the size limit. At line 122, it opens the new file for appending, but never fsyncs `self._dir`:

```python
def _rotate(self):
    if self._fd:
        self._fd.flush()
        os.fsync(self._fd.fileno())   # line 115: flushes OLD file
        self._fd.close()
    ...
    self._current_file = os.path.join(self._dir, f"{next_num:06d}.wal")
    self._fd = open(self._current_file, "ab")  # line 122: creates NEW file
    # no os.fsync on self._dir — directory entry not durable
```

If the system crashes right after rotation, the old WAL file is safely on disk, but the new WAL file's directory entry may be lost. Any records written to the new file vanish — even those that were individually fsynced at lines 128/133.

### Bitcask Segment Rotation (`hash-index-storage/bitcask.py:101-104`)

`_maybe_rotate` closes the active file and opens a new data file at `_open_active_file` (line 68). The new `.data` file is created, and subsequent writes are fsynced (line 88), but the directory entry linking `{file_id}.data` into the data directory is never fsynced.

### Bitcask Segment Rotation (`log-structured-hash-table/bitcask.py:120-127`)

Same pattern. `_open_new_segment` at line 124 and `_rotate_segment` at line 128 create new segment files. No directory fsync follows. This implementation doesn't even fsync the *file data* on writes (note the absence of `os.fsync` calls in `_write_record` at line 140), so the directory fsync gap is just one layer of a deeper durability deficit.

### LSM SSTable Flush (`log-structured-merge-tree/lsm.py:80`)

`SSTable.write` creates a new `.sst` file at line 80 (`open(path, "wb")`). The file is written and closed, but neither the file data nor the parent directory are fsynced. The directory entry for the new SSTable could be lost on crash, which would silently lose flushed memtable data — especially dangerous because the WAL is truncated (line 58-60) after flushing, so recovery would find neither the WAL nor the SSTable.

### Hint Files and Compaction Outputs

Hint files (`hash-index-storage/bitcask.py:159`, `log-structured-hash-table/bitcask.py:370`) and compaction output files (`log-structured-hash-table/bitcask.py:264`) are also created without directory fsyncs. Losing a hint file is less catastrophic (the full segment can be re-scanned), but losing a compaction output means the old segments were deleted for nothing.

## Why This Is Subtle

The gap is easy to miss because:

1. **The file's data *is* fsynced** in the better implementations (WAL, hash-index Bitcask). Developers see the `os.fsync` call and think durability is handled.
2. **It rarely manifests.** Most filesystems will flush directory metadata within seconds, so only a crash in that narrow window triggers it. This makes it nearly impossible to catch in testing.
3. **Recovery code assumes files exist.** Methods like `_wal_files` (line 82) and `_find_file_ids` (line 54 in hash-index Bitcask) use `os.listdir` to discover files. If a directory entry was lost, the file simply doesn't appear — there's no corruption signal, just silent data loss.

The fix is a small helper called after every file creation:

```python
def _fsync_directory(dir_path):
    fd = os.open(dir_path, os.O_RDONLY)
    try:
        os.fsync(fd)
    finally:
        os.close(fd)
```

This should be called in `_rotate`, `_open_active_file`, `SSTable.write`, `_write_hint_file`, and every compaction path that creates new files.

## Topics to Explore

- [general] `fsync-ordering-guarantees` — Understanding what POSIX actually guarantees about fsync vs. fdatasync, and how Linux ext4's `auto_da_alloc` partially mitigates the directory fsync gap for overwrites (but not new file creation)
- [function] `log-structured-merge-tree/lsm.py:SSTable.write` — The SSTable flush path that neither fsyncs file data nor the directory, creating a double durability gap where WAL truncation can race ahead of durable SSTable creation
- [function] `write-ahead-log/wal.py:truncate` — How WAL truncation rewrites files and removes old ones, compounding the directory fsync gap when truncation deletes a file whose replacement's directory entry wasn't fsynced
- [function] `hash-index-storage/bitcask.py:compact` — The compaction path creates merged files and deletes old segments; without directory fsync, a crash mid-compact could lose the merged output while the old segments are already gone
- [general] `rename-fsync-protocol` — The safe pattern used by databases like SQLite and PostgreSQL: write temp file, fsync file, rename into place, fsync parent directory — and how `os.rename` (line 297 in hash-index Bitcask, line 301 in log-structured Bitcask) without directory fsync is itself a durability gap

## Beliefs

- `no-directory-fsync-anywhere` — None of the 13 `os.fsync` calls in the codebase target a directory file descriptor; all fsync calls operate on data file descriptors only
- `wal-rotation-creates-unsynced-directory-entry` — `WriteAheadLog._rotate` (wal.py:122) creates a new WAL file and fsyncs its contents on each write, but never fsyncs the parent directory, so the file's directory entry can be lost on crash
- `lsm-sstable-flush-has-no-fsync` — `SSTable.write` (lsm.py:80) creates and populates a new SSTable file without any fsync — neither the file data nor the directory entry — making flushed memtable data vulnerable to loss
- `recovery-assumes-directory-entries-durable` — All recovery methods (`_wal_files`, `_find_file_ids`, `_find_existing_segments`) use `os.listdir` to discover files, meaning a lost directory entry causes silent data loss with no corruption signal
- `bitcask-rename-without-dir-fsync` — Both Bitcask implementations use `os.rename` (hash-index:297, log-structured:301) during compaction without fsyncing the parent directory afterward, leaving the rename itself non-durable

