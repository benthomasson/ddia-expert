# Topic: How LevelDB's MANIFEST file provides crash-safe version transitions; compare with the WAL crash recovery in `lsm.py` (lines 88–96) which only covers data, not metadata

**Date:** 2026-05-29
**Time:** 13:44

# MANIFEST vs. WAL: Metadata Crash Safety

## The Gap in This Implementation

The LSM tree in `log-structured-merge-tree/lsm.py` has a WAL that protects **data** — unflushed memtable entries survive a crash via `WAL.replay()` (lines 28–52). But there's no equivalent protection for **metadata**: which SSTables exist, what sequence numbers are valid, and (in a leveled scheme) what level each file belongs to.

On startup, `LSMTree._load_existing_sstables()` (lines 218–228) reconstructs metadata by scanning the directory:

```python
files = sorted(f for f in os.listdir(self._dir) if f.endswith(".sst"))
for fname in files:
    seq = int(fname.split("_")[1].split(".")[0])
    sst = SSTable(os.path.join(self._dir, fname), seq, self._sparse_interval)
    sst.load_index()
    self._sstables.append(sst)
```

This works for a simple single-level design, but it's fragile. Consider what happens during compaction: you merge several SSTables into one new file, then delete the old ones. If a crash occurs **after** creating the new file but **before** deleting the old files, recovery sees both — duplicate data with no way to know which files are authoritative. The inverse (old deleted, new not yet visible) loses data entirely.

## What LevelDB's MANIFEST Solves

LevelDB treats the set of live SSTables as a **versioned, immutable snapshot** called a `Version`. Every structural change — flush, compaction, file deletion — is recorded as a `VersionEdit` and appended to the MANIFEST file. This is essentially a **WAL for metadata**:

1. **VersionEdit** encodes a delta: "add file X at level 2 with key range [a, m]; remove files Y and Z from level 1."
2. The edit is **fsync'd to MANIFEST** before any file deletion occurs.
3. On recovery, LevelDB replays all VersionEdits from the MANIFEST to reconstruct the current Version — exactly which files are live and at what levels.
4. A `CURRENT` file (a single line) points to the active MANIFEST, enabling atomic switchover when the MANIFEST itself is rotated.

This gives crash-safe **atomic version transitions**: the old set of SSTables is valid until the VersionEdit is durable, at which point the new set is valid. There's never an ambiguous in-between.

## Comparing the Two Recovery Paths

| Concern | `lsm.py` WAL | LevelDB MANIFEST |
|---------|--------------|-----------------|
| **What it protects** | Memtable entries (key-value data) | SSTable membership, levels, key ranges |
| **Recovery method** | Replay append log into memtable | Replay VersionEdits to rebuild Version |
| **Atomicity unit** | Single key write | Entire compaction/flush (add + remove files) |
| **Corruption detection** | Truncation on partial read (line 37–40) | CRC per record block |
| **Metadata recovery** | Directory scan (`os.listdir`) | Deterministic replay |

The standalone WAL module in `write-ahead-log/wal.py` is more sophisticated — it has CRC checksums (line 36), batch commits with `OP_COMMIT` markers (lines 143–157), checkpoints (lines 159–166), and rotation (lines 105–114). But even this richer WAL only covers data operations. Neither module logs structural changes like "file `sst_007.sst` is now part of the live set."

## Why This Matters

The directory-scan approach in `_load_existing_sstables` works here because this implementation has no levels — all SSTables are peers searched newest-to-oldest. But the moment you add leveled compaction (as in `sstable-and-compaction/sstable.py`, which already has a `level` field in `SSTableMetadata` at line 39), you need to know which level each file belongs to. That information isn't in the filename or the file contents — it's metadata that lives in the MANIFEST. Without it, a crash during compaction can corrupt the level structure silently.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:_load_existing_sstables` — The directory-scan recovery strategy and its failure modes during concurrent compaction
- [file] `sstable-and-compaction/sstable.py` — Already defines `SSTableMetadata.level` but has no persistent metadata log; a natural place to add MANIFEST-style tracking
- [function] `write-ahead-log/wal.py:append_batch` — Atomic batch commits with `OP_COMMIT` — the same pattern a MANIFEST would use for atomic version transitions
- [general] `leveldb-version-set` — Study LevelDB's `version_set.cc` and `version_edit.cc` to see how VersionEdits are encoded, replayed, and how the CURRENT pointer enables MANIFEST rotation
- [function] `write-ahead-log/wal.py:checkpoint` — Checkpoint records could be extended to snapshot metadata state, similar to how MANIFEST snapshots the full Version periodically to bound replay cost

## Beliefs

- `lsm-wal-covers-data-not-metadata` — The WAL in `lsm.py` replays key-value entries into the memtable on recovery but does not record which SSTables are live; SSTable membership is reconstructed by directory scan
- `directory-scan-recovery-unsafe-during-compaction` — `_load_existing_sstables` using `os.listdir` cannot distinguish between pre-compaction and post-compaction file sets after a crash, risking duplicates or data loss
- `sstable-metadata-has-level-field-but-no-persistence` — `SSTableMetadata` in `sstable-and-compaction/sstable.py` tracks a `level` field (line 39) that would be lost on crash since no metadata log persists it
- `wal-module-has-batch-atomicity-pattern` — `WriteAheadLog.append_batch` in `write-ahead-log/wal.py` writes multiple records atomically with a trailing COMMIT marker — the same pattern needed for atomic metadata version transitions

