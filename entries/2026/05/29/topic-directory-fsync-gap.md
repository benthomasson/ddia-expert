# Topic: None of the implementations fsync the parent directory after creating new files (WAL rotation, SSTable creation); explore how this could cause data loss on ext4/XFS

**Date:** 2026-05-29
**Time:** 10:03

# Missing Parent Directory `fsync` After File Creation

## The Problem

On POSIX filesystems like ext4 and XFS, creating a new file is a two-step operation at the filesystem level:

1. The file's data and metadata are written (controlled by `fsync` on the file descriptor)
2. The **directory entry** pointing to that file is updated (controlled by `fsync` on the **parent directory**)

These are independent durability guarantees. You can `fsync` a newly created file all you want — if the machine crashes before the directory entry is persisted, the file simply doesn't exist after recovery. The data blocks may be on disk, but nothing points to them.

Every implementation in this codebase has this gap.

## WAL Rotation: `write-ahead-log/wal.py:_rotate` (line ~107)

```python
def _rotate(self):
    """Create a new WAL file."""
    if self._fd:
        self._fd.flush()
        os.fsync(self._fd.fileno())
        self._fd.close()
    # ...
    self._current_file = os.path.join(self._dir, f"{next_num:06d}.wal")
    self._fd = open(self._current_file, "ab")
```

The old WAL segment is properly fsynced before closing — good. But the new file `000002.wal` is opened via `open(..., "ab")`, which creates it, and **no `fsync` is issued on `self._dir`**. If the system crashes after writing records to the new segment but before the directory entry is flushed:

- The old WAL segment is intact (it was fsynced)
- The new WAL segment **does not exist** from the filesystem's perspective
- All records written to the new segment are lost

This is particularly dangerous because `_do_sync` (line ~117) faithfully fsyncs every write to the new file descriptor — giving the caller a false sense of durability. The `fsync` succeeds, the kernel returns, and the data is on the platter, but the directory doesn't know the file is there.

## SSTable Creation: `sstable-and-compaction/sstable.py:SSTableWriter.finish` (line ~89)

```python
def finish(self) -> SSTableMetadata:
    """Finalize the SSTable, write index and footer."""
    # ... writes index and footer ...
    self._f.close()
```

`SSTableWriter.finish` doesn't even `fsync` the file itself before closing — `close()` does not guarantee durability. But even if it did, there's no directory fsync. After a memtable flush, if the system crashes:

- The SSTable file may not exist on disk
- The WAL that was truncated (because the flush "succeeded") is gone
- The data exists nowhere

## LSM SSTable Write: `log-structured-merge-tree/lsm.py:SSTable.write` (line ~80)

```python
@staticmethod
def write(path, seq, entries, sparse_index_interval=16):
    sst = SSTable(path, seq, sparse_index_interval)
    with open(path, "wb") as f:
        # ... writes entries and footer ...
    sst._sparse_index = offsets
    return sst
```

The `with` block closes the file, but there is no `fsync` on the file descriptor and no `fsync` on the parent directory. The LSM's WAL truncation (`WAL.truncate` at line ~58) then destroys the WAL:

```python
def truncate(self):
    self._fd.close()
    self._fd = open(self._path, "wb")  # wipes WAL contents
```

This creates a window where the WAL is truncated but the SSTable doesn't durably exist. After a crash, both are gone.

## Why ext4 and XFS Specifically

This isn't theoretical. On ext4 with the default `data=ordered` mount option, file data is flushed before metadata — but **directory entries are metadata of the parent directory**, not the file. The kernel can reorder the directory update arbitrarily relative to other writes. XFS is even more aggressive about reordering metadata operations.

On ext4, the typical crash window is 5 seconds (the default `commit` interval for journaling). On XFS, it can be longer depending on log flush heuristics.

Notably, `btrfs` with its copy-on-write semantics has a similar issue — a directory fsync is required to guarantee a new file's existence after crash.

## What the Fix Looks Like

The missing operation is:

```python
dir_fd = os.open(parent_dir, os.O_RDONLY)
try:
    os.fsync(dir_fd)
finally:
    os.close(dir_fd)
```

This must be called after creating (and fsyncing) the new file. Every WAL rotation, SSTable flush, and compaction output needs this.

## The Compound Failure

The most dangerous scenario combines two gaps:

1. **SSTable created without directory fsync** — file may not survive crash
2. **WAL truncated after SSTable creation** — recovery data destroyed
3. **Crash** — SSTable gone, WAL gone, data gone

This is exactly the pattern in `lsm.py` where `SSTable.write` is followed by `WAL.truncate`. Production databases (LevelDB, RocksDB, SQLite) all fsync the parent directory after creating new files precisely to close this window.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:truncate` — Truncation removes WAL records after flush; understanding its interaction with SSTable durability is critical to the data loss scenario
- [function] `log-structured-merge-tree/lsm.py:SSTable.write` — No fsync on either the file or directory; trace how the LSMTree calls this during flush and what happens to the WAL afterward
- [general] `rename-fsync-pattern` — The atomic rename pattern (`write-to-temp, fsync, rename, fsync-parent-dir`) used by production databases like RocksDB and SQLite to guarantee both content and existence durability
- [file] `sstable-and-compaction/sstable.py` — SSTableWriter.finish also skips file-level fsync; compare with how LevelDB's `WritableFile::Sync` works
- [general] `ext4-mount-options-durability` — How `data=journal` vs `data=ordered` vs `data=writeback` mount options change the crash semantics and whether any mode makes the missing directory fsync safe (none do)

## Beliefs

- `wal-rotation-missing-dir-fsync` — `WriteAheadLog._rotate` creates a new WAL segment file but never fsyncs the parent directory, so the new segment may not exist after a crash even though its contents were fsynced
- `sstable-writer-no-fsync` — `SSTableWriter.finish` calls `self._f.close()` without any `fsync` on the file descriptor or parent directory, providing no durability guarantee for the written SSTable
- `lsm-wal-truncate-before-sstable-durable` — `LSMTree` truncates the WAL after `SSTable.write` returns, but since neither the SSTable file nor its directory entry are fsynced, a crash can lose both the SSTable and the WAL data
- `no-dir-fsync-anywhere` — No module in the codebase opens a directory file descriptor for `fsync`; the pattern `os.open(dir, os.O_RDONLY)` + `os.fsync` appears nowhere
- `wal-do-sync-false-durability` — `WriteAheadLog._do_sync` fsyncs the file contents in sync mode, but after rotation this provides false durability since the directory entry for the new file may not be persisted

