# Topic: The current compaction isn't atomic: a crash between deleting old segments and renaming the active segment could leave the store in an inconsistent state. Worth analyzing what guarantees are actually needed.

**Date:** 2026-05-28
**Time:** 18:56

I don't have read access to the target repo, but the observations give enough to trace the crash windows. Let me analyze the two Bitcask variants and the LSM tree.

---

# Compaction Atomicity Analysis

## The Core Problem

Compaction in all three implementations follows a **delete-then-rename** pattern: old segment files are removed, then the active segment is renamed to fill the gap. These are separate filesystem operations with no transactional guarantee. A crash between them leaves the store in a state that the recovery code doesn't expect.

## The Crash Windows

### 1. `hash-index-storage/bitcask.py` (lines 288–297)

The grep results show three sequential filesystem mutations:

- **Line 288**: `os.remove(data_path)` — deletes old data files
- **Line 290**: `os.remove(hint_path)` — deletes hint files
- **Line 297**: `os.rename(self._data_path(old_active_id), ...)` — renames the active segment

**Crash window 1** (between lines 288–290): A data file is deleted but its hint file still exists. On recovery, `_load_hint_file` (line ~153) would load index entries pointing to a file that no longer exists. Every `get()` for those keys would read from a missing file — a hard crash, not silent data loss.

**Crash window 2** (between lines 290–297): Old segments are deleted, but the active segment hasn't been renamed yet. The in-memory index was rebuilt from the old segments during the previous startup, and now those segments are gone. On recovery, `_rebuild_index` scans `_find_file_ids()` — it would find the active file at its old ID, rebuild from it, but all data that was only in the deleted old segments is **permanently lost**.

### 2. `log-structured-hash-table/bitcask.py` (lines 292–301)

Same pattern, different file:

- **Line 292**: `os.remove(seg_path)` — deletes frozen segments
- **Line 295**: `os.remove(hint)` — deletes hint files
- **Line 301**: `os.rename(old_active_path, new_active_path)` — renames active segment

The critical difference here: this variant uses CRC32 integrity checks (the `HEADER_FMT = "!III"` at line 8 includes a CRC field). Recovery via `_scan_segment` (line ~88) validates CRCs and stops scanning on corruption. This doesn't help with the atomicity problem — CRCs detect corrupt *records*, not missing *files* — but it does mean a partial write during compaction is handled gracefully.

The crash window is structurally identical to the first Bitcask. The merged output was written to a new segment, old segments were deleted, and if the process dies before the rename, the merged data exists somewhere but the active segment pointer is stale.

### 3. `log-structured-merge-tree/lsm.py` (line 353)

The LSM tree has a simpler version of the problem:

- **Line 353**: `os.remove(sst.path)` — deletes old SSTables after merge

There's no rename step visible in the observations. The LSM tree writes merged SSTables as new files (via `SSTable.write`), then deletes the inputs. If the crash happens after some but not all old SSTables are deleted, recovery would scan the surviving old SSTables *and* the new merged one, producing **duplicate entries**. Since the memtable and newer SSTables take precedence, the duplicates wouldn't cause incorrect reads — but they waste space and could confuse compaction heuristics.

## What Guarantees Are Actually Needed

The question asks what's *actually* needed, not what's ideal. Here's the hierarchy:

**Must have: No data loss.** A crash during compaction must never lose committed data. The merged output must be fully written and durable before any input is deleted.

**Must have: Recoverable state.** On restart, the recovery code must be able to reconstruct a consistent index regardless of where in the compaction sequence the crash occurred. This means recovery needs to handle:
- Merged output exists alongside old segments (duplicates — safe if reads prefer newer data)
- Old segments partially deleted (dangerous — data only exists in deleted files)

**Nice to have: No duplicate work.** Ideally recovery doesn't re-read data that was already compacted. But correctness trumps efficiency.

## The Fix Pattern

The standard approach (used by real Bitcask, LevelDB, RocksDB) is:

1. Write the merged output to new files (already done)
2. `fsync` the new files and their parent directory
3. Write a **manifest** or **compaction log** that atomically records "replace segments [A, B, C] with [D]"
4. `fsync` the manifest
5. Delete old segments (now safe — the manifest says to ignore them)
6. On recovery: read the manifest first, then only load the segments it references

The manifest write (step 3–4) is the atomic commit point. Steps before it are invisible; steps after it are idempotent retries.

None of the three implementations have a manifest. The `_find_file_ids()` / `_find_existing_segments()` discovery is purely filesystem-based, which is exactly what makes the delete-then-rename gap dangerous.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — The full compaction method (lines 194–321) to see if there's any ordering or fsync between the delete and rename steps that the grep didn't capture
- [function] `log-structured-hash-table/bitcask.py:_recover` — How recovery (line ~69) rebuilds state from segments, and whether it can tolerate finding both old and merged segments simultaneously
- [general] `manifest-based-compaction` — How LevelDB's MANIFEST and RocksDB's MANIFEST track compaction state atomically, and what the minimal equivalent would look like for these implementations
- [function] `log-structured-merge-tree/lsm.py:compact` — The full LSM compaction (line 319+) to see whether it writes new SSTables before or after deleting old ones, and whether the SSTable metadata list is updated atomically
- [file] `sstable-and-compaction/sstable.py` — The `_stcs_compact` (line 357) and `_lcs_compact` (line 400) methods to compare compaction strategies and whether leveled compaction has a different crash-safety profile

## Beliefs

- `bitcask-compaction-not-crash-safe` — Both Bitcask implementations (`hash-index-storage/bitcask.py`, `log-structured-hash-table/bitcask.py`) delete old segments before renaming the active segment, creating a window where committed data can be permanently lost on crash
- `no-compaction-manifest` — None of the three storage engines (hash-index, log-structured-hash-table, LSM) use a manifest or compaction log to make the segment swap atomic; segment discovery is purely filesystem-based via directory listing
- `lsm-compaction-duplicates-safe` — The LSM tree's compaction crash window produces duplicate entries rather than data loss, because newer SSTables take read precedence over older ones
- `crc-does-not-prevent-atomicity-bugs` — The CRC32 integrity checks in `log-structured-hash-table/bitcask.py` detect corrupt records within a segment but provide no protection against the cross-file atomicity problem in compaction
- `hint-file-orphan-risk` — In `hash-index-storage/bitcask.py`, a crash between deleting a data file (line 288) and its hint file (line 290) leaves an orphaned hint that directs reads to a nonexistent data file

