# Topic: `os.rename()` is atomic on POSIX but the directory entry change is only durable after fsyncing both source and destination directories

**Date:** 2026-05-29
**Time:** 08:31

# `os.rename()` Atomicity vs. Durability

## The Distinction

`os.rename()` on POSIX is **atomic** — the kernel updates the directory entries in a single operation, so concurrent readers see either the old name or the new name, never a torn state. But atomic does not mean **durable**. The rename modifies directory entries (metadata in the filesystem's directory inodes), and those changes sit in the kernel's page cache until either the OS flushes them or you explicitly call `fsync()` on the directory file descriptors.

If the machine crashes between the `os.rename()` call and the next filesystem flush, the rename may be lost entirely — the file could reappear under its old name, or in the worst case (if source and destination are different directories), the entry could be missing from both.

## What the Codebase Does

Both Bitcask implementations use `os.rename()` during compaction:

- **`hash-index-storage/bitcask.py:297`** — renames the old active data file after compaction
- **`log-structured-hash-table/bitcask.py:301`** — renames the active file path to a new path

The hash-index Bitcask does fsync file *contents* before writing (`hash-index-storage/bitcask.py:88`), which is correct for data durability. But neither implementation fsyncs the **directory** after the rename.

## What's Missing

The `find_dir_fsync` search confirms there is **zero** directory-level fsync anywhere in the codebase. The pattern `os.open.*O_RDONLY|dirfd|directory.*sync` found only regular file fsyncs. A correct durable rename looks like:

```python
os.rename(old_path, new_path)

# Fsync destination directory to make the new entry durable
dest_dir_fd = os.open(os.path.dirname(new_path), os.O_RDONLY)
try:
    os.fsync(dest_dir_fd)
finally:
    os.close(dest_dir_fd)

# If source dir differs, fsync it too to make the removal durable
if os.path.dirname(old_path) != os.path.dirname(new_path):
    src_dir_fd = os.open(os.path.dirname(old_path), os.O_RDONLY)
    try:
        os.fsync(src_dir_fd)
    finally:
        os.close(src_dir_fd)
```

In both Bitcask implementations, the source and destination are in the same directory (the data dir), so a single directory fsync would suffice.

## Why It Matters Here

The WAL implementation (`write-ahead-log/wal.py`) is careful about fsync — it calls `os.fsync()` on the file descriptor after every write in sync mode (line 115), on rotation (line 128), and on truncation (line 184). The B-tree engine (`b-tree-storage-engine/btree.py`) pairs every `flush()` with an `os.fsync()` (lines 105, 113, 137, 144, 171). These are all **file content** fsyncs.

But the Bitcask compaction renames are the one place where **directory metadata** durability matters, and it's unaddressed. In practice, this means a crash right after compaction could lose the rename, leaving the store with the pre-compaction file layout. The data itself isn't lost (the old files still exist), but the compaction work is — and if the old active file was partially overwritten or removed, recovery could get confused.

## The Broader Principle

This is a common pitfall from DDIA Chapter 3: storage engines must distinguish between three levels of guarantee:

1. **Visibility** — other processes see the change (guaranteed by kernel after `rename()` returns)
2. **Atomicity** — the change is all-or-nothing (guaranteed by POSIX `rename()`)
3. **Durability** — the change survives a crash (requires `fsync()` on the affected directories)

Most of the codebase handles (1) and (3) for file contents correctly. The gap is (3) for directory-level operations.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — The compaction routine that performs the rename; trace how it rebuilds the keydir and what happens if the rename is lost on crash
- [function] `write-ahead-log/wal.py:_rotate` — WAL rotation creates new files (a directory entry change) without fsyncing the directory — same class of bug
- [file] `log-structured-merge-tree/lsm.py` — The LSM tree writes new SSTable files during flush and compaction; check whether those new-file creations fsync the directory
- [general] `crash-recovery-invariants` — Map out the full set of crash points across all storage engines and verify which ones actually survive `kill -9` followed by immediate reboot
- [function] `b-tree-storage-engine/btree.py:_write_page` — The B-tree fsyncs page writes diligently; compare its discipline with the Bitcask compaction path

## Beliefs

- `no-directory-fsync-anywhere` — No storage engine in the repo fsyncs a directory file descriptor after `os.rename()` or file creation, leaving directory metadata changes non-durable across crashes
- `bitcask-compaction-rename-not-durable` — Both Bitcask implementations (`hash-index-storage/bitcask.py:297`, `log-structured-hash-table/bitcask.py:301`) call `os.rename()` during compaction without fsyncing the containing directory
- `wal-rotation-same-gap` — `write-ahead-log/wal.py:_rotate` creates a new WAL file and closes the old one but does not fsync the WAL directory, so the new file's directory entry is not crash-durable
- `file-content-fsync-is-thorough` — The WAL, B-tree, and Bitcask engines all correctly call `os.fsync()` on file data after writes; the durability gap is specifically at the directory-entry level

