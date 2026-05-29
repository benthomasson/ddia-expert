# Topic: What happens when a crash occurs mid-compaction? The delete-then-rename sequence has a window where data could be lost

**Date:** 2026-05-28
**Time:** 18:25

# Crash Safety During Compaction

## The Core Problem

These implementations have a fundamental crash-safety gap: **they delete old data files before the new compacted data is fully committed to a discoverable location, with no manifest or journal to track which files constitute the current valid state.** A crash in that window means the recovery logic can't find either the old data (deleted) or the new data (not yet renamed/registered).

## The Three Implementations and Their Crash Windows

### 1. `hash-index-storage/bitcask.py` — Delete, then rename

The `compact()` method (starting at line 194) follows this sequence:

1. Write live entries into a new compacted data file
2. **Delete old data files** — line 288: `os.remove(data_path)`
3. **Delete old hint files** — line 290: `os.remove(hint_path)`
4. **Rename** — line 297: `os.rename(self._data_path(old_active_id), ...)`

**Crash window:** If the process dies between step 2 and step 4, the old segments are gone. The new compacted file exists on disk, but it may not have the file ID that `_recover()` (line 112) expects when it calls `_find_file_ids()` (line 53) to scan for `*.data` files. The in-memory `keydir` index — rebuilt from disk on startup — will be missing every key that lived in the deleted segments.

Even within step 2 alone: if there are multiple immutable segments and the crash happens after deleting only some, `_rebuild_index()` (line 117) will scan the surviving files but skip the deleted ones. Keys whose latest version was in a deleted segment silently vanish.

### 2. `log-structured-hash-table/bitcask.py` — Same pattern, same gap

The `compact()` method (line 219) does:

1. Merge frozen segments into a new compacted segment
2. **Delete old segments** — line 292: `os.remove(seg_path)`
3. **Delete old hint files** — line 295: `os.remove(hint)`
4. **Rename active segment** — line 301: `os.rename(old_active_path, new_active_path)`

**Crash window:** Between lines 292 and 301. The `_recover()` method rebuilds the index by scanning whatever segment files exist on disk. If old segments are deleted but the rename hasn't happened, recovery sees a gap: segment IDs that the compacted file was supposed to replace are gone, and the compacted file may have an unexpected name or sequence number.

### 3. `log-structured-merge-tree/lsm.py` — Delete without manifest

The `compact()` method (line 319) writes a new merged SSTable, then at line 353: `os.remove(sst.path)` for each old SSTable.

This is slightly different — there's no rename step. The new SSTable is written first, then old ones are deleted. But there's **no manifest file** tracking which SSTables are current. On recovery, the LSM tree must reconstruct state from whatever files exist. If a crash happens mid-deletion:

- **Best case:** Some old SSTables survive alongside the new one, causing duplicate entries (safe but wasteful — reads see duplicates that merge-iterate resolves).
- **Worst case:** An old SSTable is deleted but the new SSTable wasn't fully fsynced. The data in that old SSTable is gone and the new file may be truncated or corrupt.

## Why This Matters: The Missing Safeguard

The safe compaction protocol (used by production systems like LevelDB/RocksDB) is:

1. Write new compacted file(s) to temporary names
2. `fsync` the new files
3. **Atomically update a manifest** (e.g., a `CURRENT` or `MANIFEST` file) that lists the active file set
4. `fsync` the manifest directory entry
5. Delete old files — now safe, since they're no longer referenced

None of these implementations have a manifest. The `_recover()` / `_rebuild_index()` methods discover files by listing the directory (`os.listdir`), so the directory listing *is* the implicit manifest. But directory state isn't transactional — there's no way to atomically say "these 5 files are out, this 1 file is in."

The hint files (`_hint_path`, `_load_hint_file`) speed up recovery but don't solve the consistency problem — they're an optimization for the index rebuild, not a transactional record of which data files are valid.

## What a Fix Would Look Like

A manifest file (sometimes called `CURRENT`) that lists every active SSTable/segment ID. Compaction becomes:

1. Write new compacted files
2. Write new manifest listing the new file set (atomic via write-to-temp + `os.replace`)
3. Delete old files

On crash recovery, read the manifest to know which files to load. Orphaned files (written but not in manifest) get cleaned up. Deleted-but-still-in-manifest files indicate a bug, not a crash, and can be detected.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:_recover` — The recovery path that must make sense of whatever files survive a crash; understanding its assumptions reveals exactly which crash states are unrecoverable
- [function] `log-structured-merge-tree/lsm.py:compact` — The LSM compaction has a different (arguably safer) ordering than the Bitcask variants; compare the delete-last approach to the Bitcask delete-then-rename approach
- [file] `write-ahead-log/wal.py` — The WAL implementation (line 62) handles crash recovery with CRC validation and partial-write detection — techniques that could be adapted to make compaction crash-safe
- [general] `manifest-based-compaction` — How LevelDB's MANIFEST and RocksDB's version sets solve exactly this problem with atomic state transitions
- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — Shows how partial writes at segment boundaries are handled (CRC check at line ~105 stops scanning on corruption), but this tolerance doesn't extend to missing files

## Beliefs

- `no-compaction-manifest` — None of the three storage engines (hash-index-storage, log-structured-hash-table, log-structured-merge-tree) use a manifest file to track which data files are current; file discovery relies entirely on `os.listdir` directory scanning
- `delete-before-rename-ordering` — Both Bitcask implementations delete old segment files (`os.remove`) before renaming the replacement, creating a window where neither old nor properly-named new data is on disk (hash-index-storage/bitcask.py:288-297, log-structured-hash-table/bitcask.py:292-301)
- `hint-files-are-optimization-only` — Hint files accelerate index rebuild during recovery but carry no crash-consistency guarantees; they are not consulted to determine which data files are valid
- `lsm-compaction-deletes-without-fsync-barrier` — The LSM tree's `compact()` calls `os.remove(sst.path)` at line 353 without an explicit `fsync` on the new SSTable or its parent directory beforehand, meaning the new file's data may not be durable when the old file is unlinked
- `index-is-volatile` — All three engines rebuild their in-memory index (`keydir`, `_index`) entirely from disk files on startup; a crash that leaves files in an inconsistent state produces a silently incomplete index with no error

