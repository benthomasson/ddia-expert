# Topic: The compaction sequence is not atomic; worth analyzing which failure points leave recoverable vs corrupted state

**Date:** 2026-05-29
**Time:** 06:46

I don't have read permissions to the target repository, so I'll work from the observation data. Notably, the observations cut off at line 200 for every file, but the compaction methods all start *after* line 200 (except `hash-index-storage/bitcask.py:compact` at line 194, where we get only the preamble). The grep results give us enough to reconstruct the critical failure sequence.

---

# Compaction Atomicity Analysis

## The Core Problem

Compaction across all four implementations follows a multi-step sequence — write new file, update in-memory index, delete old files — with no transactional wrapper. A crash at any intermediate step leaves the system in a state that may or may not be recoverable depending on *where* in the sequence the failure occurs.

## The Three Compaction Implementations

### 1. Bitcask (`hash-index-storage/bitcask.py`)

The compaction method begins at line 194. From the grep results, we can trace the critical sequence:

- **Line 288**: `os.remove(data_path)` — deletes old data file
- **Line 290**: `os.remove(hint_path)` — deletes old hint file
- **Line 297**: `os.rename(self._data_path(old_active_id), ...)` — renames a file

The danger here is the **delete-before-rename** ordering. If the process crashes between line 288 (old data deleted) and line 297 (rename), the data referenced by the in-memory index is gone. On restart, `_recover()` (line 67) rebuilds the index by scanning existing segments, so it won't point to deleted files — but any keys that existed *only* in the deleted segment and hadn't yet been written to the new compacted file are permanently lost.

The `_recover` method (lines 67–87) is the safety net: it rebuilds from whatever files survive on disk. But it assumes all segment files are complete and internally consistent. There's no checkpointing of "which segments participated in this compaction round."

### 2. Bitcask Variant (`log-structured-hash-table/bitcask.py`)

Same pattern, different line numbers:

- **Line 292**: `os.remove(seg_path)` — delete old segment
- **Line 295**: `os.remove(hint)` — delete old hint
- **Line 301**: `os.rename(old_active_path, new_active_path)` — rename

This is structurally identical to the first, with the same vulnerability window between delete and rename.

### 3. LSM Tree (`log-structured-merge-tree/lsm.py`)

The `compact()` method starts at line 319. The key operation:

- **Line 353**: `os.remove(sst.path)` — delete old SSTable

The LSM tree compaction merges multiple SSTables into one. The sequence is: merge-sort entries from input SSTables → write a new SSTable → replace the SSTable list → delete old SSTables. The `os.remove` at line 353 happens inside what's likely a loop over the compacted-away SSTables.

### 4. SSTable Manager (`sstable-and-compaction/sstable.py`)

Both size-tiered (`_stcs_compact`, line 357) and leveled (`_lcs_compact`, line 400) compaction strategies exist. The `run_compaction` dispatcher is at line 325. We can't see the implementation body, but the `SSTableWriter.finish()` method (lines 84–99) is visible and reveals an important detail: **the header is written twice** — once at the start with a placeholder count (line 48), and once at the end with the real count (lines 95–96). A crash between these two writes leaves a file with `entry_count=0` in the header but actual data on disk.

## Failure Point Matrix

| Failure Point | State After Crash | Recoverable? |
|---|---|---|
| **During compacted file write** | Partial new file on disk, old files intact | **Yes** — orphaned partial file is ignored on recovery since it's not in any index; old files are replayed |
| **After new file write, before index update** | Complete new file + old files both on disk | **Yes** — recovery rebuilds from all files; duplicates resolved by newest-wins |
| **After index update, before old file delete** | New file on disk, old files still present, index points to new | **Yes** — wasted space but correct; old files are dead weight |
| **Mid-way through old file deletion** (e.g., between lines 288 and 290 in `hash-index-storage`) | Some old files deleted, some remain | **Mostly yes** — if the *new* compacted file is complete, recovery rebuilds correctly; the remaining old files add redundant data |
| **After old data file delete, before rename** (between lines 288 and 297 in `hash-index-storage`) | Old data gone, rename hasn't happened | **Data loss risk** — depends on whether the compacted output is already fully written and reachable |
| **SSTableWriter crash between header writes** (lines 48 vs 95 in `sstable.py`) | File has `entry_count=0` but real data follows | **Corrupt** — `SSTableReader.__init__` trusts the header count; the file reads as empty despite containing data |

## What Makes This Survivable (Mostly)

The recovery methods in both Bitcask implementations (`_recover` at line 67, `_scan_segment` at line 93) are **tolerant of partial writes**:

```python
# hash-index-storage/bitcask.py, lines 93-106
def _scan_segment(self, seg_path: str):
    while True:
        ...
        if len(header_data) < HEADER_SIZE:
            break  # end of file or partial write
        ...
        if len(payload) < key_size + value_size:
            break  # partial write, skip
        expected_crc = zlib.crc32(payload) & 0xFFFFFFFF
        if crc != expected_crc:
            break  # corrupted record at end, stop scanning
```

CRC validation (line 104) catches corrupted tail records. The scanner stops at the first bad record, which means partial writes from a crash are silently truncated — correct behavior.

## What's Missing From These Implementations

1. **No `fsync` on compacted output**: The `hash-index-storage` variant calls `os.fsync()` for normal writes (line ~90 area from the observation, seen in `_write_record`), but we don't see `fsync` in any compaction path. This means the OS can reorder the write of the new file and the delete of the old file — the delete can hit disk first.

2. **No compaction manifest**: Production systems like LevelDB write a `MANIFEST` file that records which SSTables are live. This makes compaction atomic at the metadata level — you write the new manifest pointing to the new files, then `fsync` it, then delete old files. None of these implementations have that.

3. **No write-ahead-log for the compaction itself**: The WAL in `lsm.py` (lines 1–60) protects the memtable, not the compaction. A crash during compaction has no WAL to tell recovery "compaction was in progress, here's what was being merged."

## The Observations Are Insufficient For

The compaction method bodies are all beyond the 200-line read window. To complete this analysis, you'd need to read:
- `hash-index-storage/bitcask.py` lines 194–321 (full `compact()`)
- `log-structured-hash-table/bitcask.py` lines 219–390 (full `compact()`)
- `log-structured-merge-tree/lsm.py` lines 319–359 (full `compact()`)
- `sstable-and-compaction/sstable.py` lines 325–438 (all compaction strategies)

The ordering of write-new / update-index / delete-old within each method is the critical detail, and we're inferring it from grep hits rather than reading it directly.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — The full compaction body (lines 194–321) to verify the exact ordering of write, index update, rename, and delete operations
- [function] `log-structured-merge-tree/lsm.py:compact` — LSM compaction (lines 319–359) to check whether the SSTable list swap and old file deletion have any ordering guarantees
- [function] `sstable-and-compaction/sstable.py:_lcs_compact` — Leveled compaction (line 400) is the most complex strategy and likely has the most failure modes due to cross-level file selection
- [general] `fsync-guarantees-across-implementations` — Audit which implementations call `fsync` after compaction writes vs only on normal writes, since this determines whether crash recovery is actually safe or just lucky
- [file] `log-structured-merge-tree/lsm.py` — The WAL class (lines 1–60) protects memtable flushes but not compaction; worth understanding the gap between what's protected and what isn't

## Beliefs

- `compaction-no-manifest` — None of the four compaction implementations use a manifest or journal file to make the "switch from old files to new files" step atomic; they rely on in-memory index updates that are lost on crash
- `recovery-tolerates-partial-writes` — Both Bitcask `_scan_segment` implementations stop scanning at the first short read or CRC mismatch, silently truncating incomplete tail records from crash-interrupted writes
- `sstable-header-written-twice` — `SSTableWriter.finish()` writes the entry count to the header at position 0 after writing all data (line 95), meaning a crash before that seek-and-rewrite leaves a structurally valid file with `entry_count=0`
- `delete-before-rename-in-bitcask` — In `hash-index-storage/bitcask.py`, `os.remove()` of old data files (line 288) precedes `os.rename()` (line 297), creating a window where a crash can leave the system with neither old nor properly-named new data
- `no-fsync-on-compaction-output` — The compaction write paths do not call `os.fsync()` before deleting old files, allowing the OS to persist the deletes before the new file's data reaches disk

