# Topic: On Linux (ext4, XFS) and macOS (APFS), directory fsync has different guarantees; explore when `os.fsync(dir_fd)` is actually necessary for rename durability

**Date:** 2026-05-28
**Time:** 18:53

# Directory Fsync and Rename Durability

## The Gap in This Codebase

Every `os.fsync()` call in this codebase operates on a **file descriptor** — never on a directory. The grep for `dir_fd` and `O_DIRECTORY` returned zero matches. Meanwhile, two Bitcask implementations perform `os.rename()` without any directory fsync afterward:

- `hash-index-storage/bitcask.py:297` — renames during compaction
- `log-structured-merge-tree/bitcask.py:301` — same pattern

This means the codebase has a **latent durability bug on Linux** that is masked on macOS.

## Why Directory Fsync Matters

`os.rename()` is atomic at the VFS level — either the old name or the new name exists, never neither, never both. But **atomic does not mean durable**. The rename modifies the directory entry (a metadata operation on the *directory*, not the file), and that directory metadata lives in its own set of disk blocks. If the system crashes before those blocks reach stable storage, the rename can be lost — the file reverts to its old name even though the data inside it was fsynced.

The correct sequence for a durable rename is:

```python
os.fsync(file_fd)            # flush file contents
os.rename(old_path, new_path)
dir_fd = os.open(dir_path, os.O_RDONLY | os.O_DIRECTORY)
os.fsync(dir_fd)             # flush directory metadata
os.close(dir_fd)
```

## Filesystem-Specific Behavior

### Linux ext4

ext4 in its default `data=ordered` mode flushes file data before committing metadata, but **does not guarantee that directory entries are persisted on rename without an explicit `fsync()` on the directory**. The journal protects filesystem consistency (no corruption), but not durability of recent operations. A crash can silently undo a rename.

If `auto_da_alloc` is enabled (default since ~2.6.30), ext4 *does* add an implicit barrier for the specific pattern of `open(O_TRUNC) + write + close + rename` (the classic safe-save), but this only covers that narrow case. A rename of a data file during compaction — exactly what `hash-index-storage/bitcask.py:297` does — is **not covered** by `auto_da_alloc`.

### Linux XFS

XFS is stricter by default. It delays metadata updates aggressively and makes no implicit durability promises for renames. Directory fsync is **required** for rename durability. Without it, a crash can lose the rename and, in older kernels, even the file contents if the file was newly allocated.

### macOS APFS

APFS uses a copy-on-write design where metadata and data changes are committed together in atomic "container superblock" updates. In practice, APFS provides **rename durability without directory fsync**. The `fsync()` syscall on macOS maps to `F_FULLFSYNC` (or `F_BARRIERFSYNC` on newer versions), and the filesystem's transaction model means directory entries are updated atomically with the rest of the checkpoint.

This is why the bug hides on macOS development machines and only manifests on Linux production deployments.

## What's Missing in the Codebase

### 1. Bitcask Compaction (Critical)

In `hash-index-storage/bitcask.py`, the `compact()` method writes new merged data files and renames the old active file (line 297). The file data is fsynced via `_write_record` (line 88), but the directory entry change from `os.rename()` is not fsynced. On ext4/XFS, a crash after the rename but before the next journal commit could lose the compaction — or worse, leave the old and new filenames in an inconsistent state relative to what `_find_file_ids()` expects on recovery.

### 2. WAL Rotation (Moderate)

In `write-ahead-log/wal.py`, the `_rotate()` method (line 112) closes the old WAL file after fsyncing it, then creates a new one. The new file's directory entry is not fsynced. If the system crashes, the new WAL file might not appear in the directory listing, and `_wal_files()` would miss it on recovery.

### 3. SSTable Writes (Moderate)

`sstable-and-compaction/sstable.py` writes SSTables via `SSTableWriter.finish()` but never fsyncs the file data (no `os.fsync()` at all in the writer) *or* the directory. The LSM tree in `log-structured-merge-tree/lsm.py` has the same gap — `SSTable.write()` creates files without fsyncing data or directory.

## The Fix Pattern

A reusable helper that the entire codebase could share:

```python
def durable_rename(old_path, new_path):
    os.rename(old_path, new_path)
    dir_fd = os.open(os.path.dirname(new_path), os.O_RDONLY)
    try:
        os.fsync(dir_fd)
    finally:
        os.close(dir_fd)
```

For new file creation (WAL rotation, SSTable writes), the same directory fsync is needed after the file is created and its data fsynced.

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — The compaction workflow is the highest-risk site for rename durability; trace the full lifecycle of old-to-new file transitions
- [function] `write-ahead-log/wal.py:_rotate` — WAL rotation creates new files without directory fsync; understand how recovery in `_recover_seq_num` would behave if the new file's directory entry was lost
- [file] `log-structured-merge-tree/lsm.py` — The LSM WAL class has no fsync at all (not even on file data); compare its durability posture to the standalone WAL implementation
- [general] `write-to-temp-then-rename-pattern` — None of the implementations use the atomic safe-save pattern (write temp, fsync, rename); explore whether compaction and SSTable creation should adopt it
- [general] `fdatasync-vs-fsync-tradeoffs` — `os.fdatasync()` skips inode metadata (mtime/size) and is faster; explore where the codebase could use it instead of `os.fsync()` without sacrificing correctness

## Beliefs

- `no-directory-fsync-anywhere` — No implementation in the codebase calls `os.fsync()` on a directory file descriptor; all 13 fsync calls target data file descriptors only
- `rename-without-barrier` — Both Bitcask compaction paths (`hash-index-storage/bitcask.py:297`, `log-structured-merge-tree/bitcask.py:301`) perform `os.rename()` without a subsequent directory fsync, making the rename non-durable on ext4 and XFS
- `apfs-masks-linux-bugs` — macOS APFS provides implicit rename durability through its CoW transaction model, so the missing directory fsync is a latent bug that only surfaces on Linux filesystems
- `lsm-wal-has-no-fsync` — The WAL class in `log-structured-merge-tree/lsm.py` calls `flush()` but never `os.fsync()`, making it weaker than the standalone WAL which offers configurable sync modes

