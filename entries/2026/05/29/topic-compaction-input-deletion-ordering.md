# Topic: In both Bitcask implementations, identify when old segment files are deleted relative to the new compacted file being committed — this ordering determines whether data loss is possible

**Date:** 2026-05-29
**Time:** 08:28

The observations cover lines 1–200 of each file but the `compact()` methods start at line 194 and 219 respectively, so most of the compaction logic is missing. However, I can still extract the critical ordering from the grep results. Let me explain what we know and what's missing.

---

## Compaction Ordering and Data Loss Risk

### What the grep results reveal

The line numbers from the grep results expose the **operation ordering** within each `compact()` method:

**`hash-index-storage/bitcask.py`** (compact starts at line 194):
1. Line 288: `os.remove(data_path)` — delete old segment data file
2. Line 290: `os.remove(hint_path)` — delete old segment hint file
3. Line 297: `os.rename(self._data_path(old_active_id), ...)` — rename something (likely the active file)

**`log-structured-hash-table/bitcask.py`** (compact starts at line 219):
1. Line 292: `os.remove(seg_path)` — delete old segment file
2. Line 295: `os.remove(hint)` — delete old hint file
3. Line 301: `os.rename(old_active_path, new_active_path)` — rename the active segment

### The critical question

In both implementations, **old segments are deleted (lines 288–290, 292–295) before a rename operation (lines 297, 301)**. The safety of compaction depends on what happens *before* those deletes — specifically:

1. Is the new compacted file fully written and fsynced to disk before old files are removed?
2. Is the in-memory index updated to point to the new compacted file before the old files are removed?

If a crash occurs after `os.remove` deletes an old segment but before the new compacted file is durable, **data in that segment is permanently lost**.

### What we can infer from the first 200 lines

Looking at the `_write_record` implementations:

- **hash-index-storage** (line 93–101): calls `flush()` and conditionally `os.fsync()` after each record — so individual writes are durable if `sync_writes=True`.
- **log-structured-hash-table** (line 161–168): calls `flush()` but **never calls `fsync()`** — data could be in the OS page cache but not on disk.

This tells us the log-structured variant is already more vulnerable: even during normal writes, a crash after `flush()` but before the OS flushes to disk loses data. During compaction, if the new compacted file isn't fsynced before old segments are deleted, the window for data loss is wider.

### What's missing (lines 200–310 of both files)

I cannot see the full `compact()` body from the observations provided. To definitively answer the question, I need to see:

- **When the new compacted segment file is opened and written** — does it use a temporary name or write directly to the final path?
- **Whether `fsync()` is called on the new file** before old segments are deleted
- **Whether `fsync()` is called on the directory** after file creation (required on Linux for the directory entry to be durable)
- **When `self.keydir` / `self._index` is updated** — before or after deletes?
- **What the `os.rename` at lines 297/301 actually renames** — is this renaming the compacted file into place (atomic commit), or renaming the active file to a new ID?

The `os.rename` calls coming *after* the `os.remove` calls is concerning — if the rename is the "commit point" for compaction, then old files are being deleted before the compaction is committed, which is a data loss window. In a safe implementation, the order should be:

1. Write new compacted file
2. `fsync` the new file
3. Update the in-memory index
4. Delete old segment files
5. `fsync` the directory

### Neither implementation fsyncs the directory

Even from what we can see, neither implementation calls `os.fsync()` on the directory file descriptor after creating new files. On Linux/ext4, this means a crash could leave the directory in a state where the new file's directory entry doesn't exist, even if the file data is on disk. This is a known subtlety that most educational implementations skip.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — Read lines 194–310 to see the full compaction flow: when the new file is written, fsynced, and when the index is updated relative to old file deletion
- [function] `log-structured-hash-table/bitcask.py:compact` — Same analysis for the log-structured variant; compare whether it has the same ordering risks or different ones
- [general] `crash-recovery-testing` — Write a test that kills the process mid-compaction (after delete, before rename) and verify whether the store can recover without data loss
- [general] `fsync-guarantees` — Investigate whether either implementation provides true durability guarantees, given that the log-structured variant never calls fsync and neither fsyncs the directory
- [file] `log-structured-hash-table/bitcask.py` — The `_frozen_segment_paths` method (referenced at line ~178 in `put`) determines which segments are eligible for compaction — understanding this boundary is key to knowing what's at risk during a crash

---

## Beliefs

- `hash-index-deletes-before-rename` — In `hash-index-storage/bitcask.py`, old segment files are deleted (line 288) before the `os.rename` call (line 297), creating a window where data exists in neither old nor new files if a crash occurs between them
- `log-structured-no-fsync-on-write` — `log-structured-hash-table/bitcask.py:_write_record` calls `flush()` but never `fsync()`, so writes are not guaranteed durable even during normal operation
- `hash-index-conditional-fsync` — `hash-index-storage/bitcask.py:_write_record` only calls `os.fsync()` when `sync_writes=True` (default), making durability a configuration choice
- `neither-impl-fsyncs-directory` — Neither Bitcask implementation calls `fsync()` on the directory file descriptor after creating new files, meaning directory entries may not survive a crash on Linux filesystems

