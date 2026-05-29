# Topic: LevelDB's `MANIFEST` / `CURRENT` file pattern solves the compaction atomicity gap; understanding it clarifies what these implementations trade away for simplicity

**Date:** 2026-05-29
**Time:** 08:50

Now I have a thorough picture. Let me write the explanation.

---

# LevelDB's MANIFEST / CURRENT Pattern and What These Implementations Trade Away

## The Problem: Compaction Is Multi-Step, Crashes Are Instantaneous

Every LSM-tree must periodically compact SSTables — merge several sorted files into fewer, larger ones, then discard the inputs. This involves at least three sequential steps:

1. **Write** the merged output to a new SSTable file
2. **Update** the in-memory record of which SSTables are "live"
3. **Delete** the old input files from disk

The critical insight: **the set of live SSTable files is mutable state**, just like user data. User data gets crash protection from a WAL. But in these implementations, the SSTable file set gets no protection at all — it exists only as `self._sstables` in memory, which vanishes on crash.

This is the compaction atomicity gap. A crash at step 2 leaves the process with both old and new files on disk and no record of which constitute the database.

## How LevelDB Solves It: MANIFEST as a WAL for Metadata

LevelDB introduces a **second write-ahead log** — not for user key-value pairs, but for metadata about which files exist. This is the MANIFEST file.

Every time the file set changes (a memtable flushes to a new SSTable, a compaction replaces inputs with outputs), LevelDB writes a **VersionEdit** record to the MANIFEST:

```
VersionEdit: { add: [sst_042.ldb @ level 1], remove: [sst_019.ldb, sst_023.ldb] }
```

This record is checksummed and appended atomically — exactly the same durability technique the WAL uses for user data. The MANIFEST append is the **commit point** for the compaction. Everything before it is tentative; everything after it is cleanup.

The parallel structure is exact:

| User data path | Metadata path |
|---|---|
| WAL protects memtable writes | MANIFEST protects file set changes |
| WAL replay recovers unflushed data | MANIFEST replay reconstructs which files are live |
| WAL uses checksummed, append-only records | MANIFEST uses the same format |

### The CURRENT File: One Level of Indirection

The MANIFEST itself can grow large and get rotated. LevelDB uses a tiny file called `CURRENT` that contains a single line — the filename of the active MANIFEST. Recovery reads `CURRENT` first, then replays the MANIFEST to reconstruct the complete file set.

Updating `CURRENT` uses the atomic write-to-temp-then-rename pattern: write the new MANIFEST name to a temp file, `fsync`, then `rename` over `CURRENT`. This makes the MANIFEST rotation itself crash-safe.

### Immutable Versions for Concurrent Reads

RocksDB extends this with **VersionSet** — each `Version` is an immutable snapshot of the live file set. A compaction creates a *new* Version by applying a VersionEdit to the current one. Readers hold a reference to the old Version and continue reading from it while the new Version takes effect. The old Version is reference-counted and freed when the last reader finishes.

This eliminates the concurrency hazard entirely: no reader ever sees a half-updated file list.

## What These Implementations Do Instead

### LSM Tree (`log-structured-merge-tree/lsm.py`)

The `compact()` method (starting around line 319) performs a merge-all compaction:

1. Reads all entries from all SSTables via `scan_all()`
2. Sorts by `(key, -seq)`, deduplicates, drops tombstones
3. Writes a single new SSTable via `SSTable.write()`
4. Replaces `self._sstables = [new_sst]` (in-memory only)
5. Deletes old files via `os.remove(sst.path)` (line ~353)

**No MANIFEST, no CURRENT, no atomic commit point.** Step 4 is the "commit" but it's volatile — lost on crash. Recovery in `_load_existing_sstables()` scans the directory for `.sst` files and loads whatever it finds. After a crash mid-compaction, it finds both old and new files with no way to distinguish them.

The data isn't *lost* — newest-wins dedup at read time handles the duplicates. But the system has no way to know compaction was in progress, can't clean up orphaned files, and wastes space indefinitely.

### Bitcask (`hash-index-storage/bitcask.py`)

Compaction follows a **delete-before-rename** pattern that's more dangerous:

- **Line 288**: `os.remove(data_path)` — deletes old data files
- **Line 290**: `os.remove(hint_path)` — deletes hint files  
- **Line 297**: `os.rename(...)` — renames the active segment

A crash between lines 288 and 297 can permanently lose data: the old segments are deleted, but the compacted output isn't yet at its final path. Recovery via `_recover()` rebuilds the index from whatever files exist on disk — but the deleted files are gone.

### SSTable Compaction (`sstable-and-compaction/sstable.py`)

Both size-tiered (`_stcs_compact`, line 357) and leveled (`_lcs_compact`, line 400) strategies exist. The `SSTableWriter` has an additional vulnerability: it writes the header with `entry_count=0` at file creation (line ~48), then seeks back to update it with the real count in `finish()` (line ~95). A crash between these writes leaves a structurally valid file that reads as empty.

### WAL Truncation (`write-ahead-log/wal.py`)

The WAL truncation — the one component whose entire purpose is crash safety — is itself not crash-safe. `truncate()` (around line 203) opens WAL files with `"wb"` (which instantly zeros the file) and rewrites kept records. A crash mid-rewrite loses both the old and new content. No temp-file-then-rename pattern is used.

## What's Traded Away

These implementations make deliberate (or at least consistent) simplifications:

| Capability | LevelDB | These Implementations |
|---|---|---|
| Persistent file set tracking | MANIFEST (append-only log) | In-memory `self._sstables` list only |
| Atomic metadata transitions | VersionEdit + fsync | Direct mutation of Python list |
| Crash-safe file creation | Write temp → fsync → rename | Write directly to final path |
| Directory durability | `fsync(dir_fd)` after rename | No directory fsync anywhere |
| Concurrent read safety | Immutable Version snapshots | Mutable list, no locking |
| Compaction recovery | Replay MANIFEST to exact state | Directory scan + guess |

The tradeoffs are reasonable for educational code: the MANIFEST pattern adds ~300 lines of bookkeeping that obscures the core LSM algorithms. But the consequence is that **every file-mutating operation — flush, compaction, WAL truncation — has a crash window where data can be lost or state can become inconsistent**.

The WAL protects user writes. Nothing protects the metadata about where those writes end up.

## Topics to Explore

- [general] `leveldb-version-edit-format` — Study LevelDB's `VersionEdit::EncodeTo` in `db/version_edit.cc` to see the compact binary format for add-file/remove-file records and how it mirrors WAL record framing
- [function] `log-structured-merge-tree/lsm.py:_flush` — The flush path triggers compaction and truncates the WAL; trace the ordering to see whether a crash between SSTable write and WAL truncation can lose data
- [file] `sstable-and-compaction/sstable.py` — `SSTableMetadata` has a `level` field (line ~39) that suggests level-aware compaction was intended, but without a MANIFEST the level assignments are ephemeral
- [general] `mvcc-version-lifetime` — How RocksDB's reference-counted `SuperVersion` lets iterators hold a consistent snapshot during compaction, solving the concurrency problem that mutable `self._sstables` creates
- [function] `write-ahead-log/wal.py:truncate` — The WAL's own truncation is the most ironic crash-safety gap; understanding it shows why the write-to-temp-then-rename pattern matters everywhere, not just for user data

## Beliefs

- `lsm-no-manifest` — The LSM tree has no persistent metadata log; the set of live SSTable files exists only in the in-memory `self._sstables` list and is reconstructed by directory scanning on recovery, making it impossible to distinguish completed compactions from interrupted ones
- `compaction-not-atomic` — `LSMTree.compact()` performs write-new / swap-list / delete-old as separate steps with no durable commit point; a crash between any two steps leaves the on-disk state ambiguous
- `wal-protects-data-not-metadata` — The WAL in `lsm.py` and `write-ahead-log/wal.py` records user key-value mutations but not SSTable-level state transitions (flush, compaction, level assignment), leaving metadata unprotected across crashes
- `no-atomic-file-creation` — No SSTable writer, compaction routine, or WAL truncation in any implementation uses the write-temp/fsync/rename pattern; all write directly to the final file path, exposing half-written files on crash
- `delete-before-rename-in-bitcask` — In `hash-index-storage/bitcask.py`, `os.remove()` of old data files (line 288) precedes `os.rename()` (line 297), creating a crash window where committed data can be permanently lost

