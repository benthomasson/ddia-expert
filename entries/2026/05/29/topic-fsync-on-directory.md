# Topic: Even with `fsync` on the hint file, the directory entry must also be fsynced (`os.fsync(os.open(dir, os.O_RDONLY))`) for the rename to be durable on Linux — explore whether the data file writes have the same gap

**Date:** 2026-05-29
**Time:** 08:52

I'll work from the detailed observations already gathered. Here's the analysis:

---

# Directory Fsync Gap: Hint Files and Data Files

## The Core Problem

On Linux (and most POSIX systems), `fsync` on a file descriptor guarantees that the file's **contents** are durable on disk. But a file's **directory entry** — the metadata that maps a filename to its inode — lives in a separate structure: the parent directory. When you create, rename, or delete a file, the directory itself is modified, and that modification sits in the kernel's page cache until either the OS flushes it or you explicitly fsync the directory.

The correct pattern after a rename is:

```python
os.rename(old_path, new_path)
dir_fd = os.open(os.path.dirname(new_path), os.O_RDONLY)
os.fsync(dir_fd)
os.close(dir_fd)
```

**No implementation in this repository does this.**

## Hint Files: Known Gap, Tolerable Risk

Both Bitcask implementations write hint files during compaction as a startup-time optimization. The hint file provides a shortcut to rebuild the in-memory index without scanning every data record.

In `hash-index-storage/bitcask.py`, `_write_hint_file()` writes binary records directly to the final path — no temp file, no fsync on the file contents, and no directory fsync. In `log-structured-hash-table/bitcask.py`, `create_hint_files()` follows the same pattern.

The saving grace: **hint file loss is semantically safe**. If the hint file is missing or corrupt on recovery, the engine falls back to scanning the data files directly. This is slower but correct. The directory entry not being durable is therefore a performance concern, not a correctness concern.

## Data Files: The Same Gap Exists — and It's Worse

The data file writes have the **same directory fsync gap**, but with higher consequences:

### 1. Active Data File Appends

In `hash-index-storage/bitcask.py`, `_write_record()` (around line 88) does flush + conditional `os.fsync()` on the file contents:

```python
self.active_file.flush()
if self.sync_writes:
    os.fsync(self.active_file.fileno())
```

This makes the *contents* durable. But the first write to a newly created active file also creates a new directory entry. That directory entry is never fsynced. If the system crashes after the first `os.fsync()` on the file but before the OS flushes the directory, the file's data is on disk but the **filename mapping is lost** — the OS can't find the file.

In `log-structured-hash-table/bitcask.py`, `_write_record()` (around line 165) doesn't even call `os.fsync()` — only `flush()`. So both the contents and the directory entry are vulnerable.

### 2. Segment Rotation

When the active segment exceeds `_max_segment_size`, a new segment file is created via `_open_new_segment()`. This creates a fresh directory entry that is never fsynced. A crash after writing records to the new segment but before the directory flushes means those records are in an orphaned inode — data exists on disk but is unreachable by filename.

### 3. Compaction Renames

Both Bitcask implementations use `os.rename()` during compaction (around line 297 in both). The rename atomically updates the directory to point the old filename at the new inode (or vice versa). But without a directory fsync:

- The rename can **revert** on crash — the directory reverts to pointing at the old file
- Compacted data files that were deleted via `os.remove()` can **reappear** — the deletion metadata was only in the page cache
- The in-memory index now references file IDs and offsets that don't match the on-disk state

### 4. WAL Rotation and SSTable Creation

In `write-ahead-log/wal.py`, `_rotate()` (line 115) fsyncs the old segment's contents before closing it, but doesn't fsync the directory after creating the new segment file. The new WAL segment's directory entry could be lost.

In `log-structured-merge-tree/lsm.py`, `_flush()` (line 80) writes an SSTable directly to its final path and then deletes the WAL — without fsyncing either the SSTable's directory entry or the directory change from the WAL deletion. If the system crashes after the WAL is deleted from the directory cache but before the SSTable's directory entry is flushed, **both are lost**.

## Why This Matters More for Data Files Than Hints

| Scenario | Hint File | Data File |
|----------|-----------|-----------|
| Directory entry lost after crash | Startup slower (full scan) | **Data unreachable or lost** |
| Rename reverts after crash | Stale hints (rebuild on next compact) | **Index/disk mismatch, corrupt reads** |
| Deletion reverts after crash | Extra hint file (harmless) | **Duplicate records, space leak** |

The hint file gap is a known, acceptable trade-off. The data file gap is the same bug class but with data-loss consequences.

## What a Fix Would Look Like

Every operation that modifies a directory — file creation, rename, delete — needs a subsequent directory fsync. A helper function would centralize this:

```python
def durable_rename(old_path, new_path):
    os.rename(old_path, new_path)
    dir_fd = os.open(os.path.dirname(new_path), os.O_RDONLY)
    try:
        os.fsync(dir_fd)
    finally:
        os.close(dir_fd)
```

And for file creation, after the first fsync of a newly created file:

```python
dir_fd = os.open(os.path.dirname(file_path), os.O_RDONLY)
try:
    os.fsync(dir_fd)
finally:
    os.close(dir_fd)
```

The cost is one extra fsync per file lifecycle event (creation, rename, deletion) — not per record write. This is a negligible overhead compared to per-record fsyncs.

---

## Topics to Explore

- [general] `atomic-write-via-temp-rename` — None of the implementations use the write-to-temp-then-rename pattern for creating new files (SSTables, compacted segments). This is the standard approach for crash-safe file creation and interacts directly with the directory fsync question.
- [function] `write-ahead-log/wal.py:_rotate` — WAL segment rotation creates a new file and closes the old one; trace the exact window where a crash could lose the new segment's directory entry while the old segment is already closed.
- [file] `log-structured-merge-tree/lsm.py` — The LSM flush path has the most dangerous variant: it deletes the WAL after writing the SSTable, with neither directory entry fsynced. Explore the full cascade of failure modes.
- [function] `b-tree-storage-engine/btree.py:WAL.commit` — The B-tree WAL has the strictest fsync discipline (per-entry fsync). Examine whether its commit protocol — data file fsync before WAL truncation — is undermined by the missing directory fsync on the truncated WAL.
- [general] `fsync-on-ext4-vs-other-filesystems` — ext4 with `auto_da_alloc` partially mitigates the rename-without-directory-fsync problem for replace-via-rename patterns, but not for new file creation. Explore which filesystem behaviors mask or expose these gaps.

## Beliefs

- `no-dir-fsync-anywhere` — No implementation in the repository fsyncs a parent directory after file creation, rename, or deletion, leaving all directory-entry mutations vulnerable to reordering on crash.
- `hint-file-loss-is-safe` — Hint file durability gaps are tolerable because recovery falls back to a full data-file scan; the hint is a pure optimization with no unique data.
- `data-file-dir-entry-same-gap-as-hint` — Data file writes suffer the identical directory-fsync gap as hint files, but with data-loss consequences rather than mere performance degradation.
- `lsm-flush-double-gap` — The LSM `_flush()` path deletes the WAL after writing an SSTable without fsyncing either directory entry, creating a window where both can be lost simultaneously.
- `no-temp-rename-pattern` — All implementations write directly to final file paths rather than using write-to-temp-then-atomic-rename, meaning a crash mid-write leaves a corrupt file at the production path.

