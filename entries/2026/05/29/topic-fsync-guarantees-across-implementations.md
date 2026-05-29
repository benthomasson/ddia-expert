# Topic: Audit which implementations call `fsync` after compaction writes vs only on normal writes, since this determines whether crash recovery is actually safe or just lucky

**Date:** 2026-05-29
**Time:** 08:41

# Fsync Audit: Compaction Writes vs Normal Writes

This audit determines which implementations ensure compaction output is durable on disk (via `fsync`) versus merely flushed to OS buffers, and whether crash recovery guarantees are real or accidental.

## Summary Table

| Implementation | Normal writes fsync? | Compaction fsync? | Crash-safe? |
|---|---|---|---|
| `write-ahead-log/wal.py` | Yes | N/A (no compaction) | **Yes** |
| `b-tree-storage-engine/btree.py` | Via WAL | N/A (page-based, no compaction) | **Yes** |
| `hash-index-storage/bitcask.py` | Conditional (`sync_writes`) | **Unknown** (code cut off) | **Maybe** |
| `log-structured-merge-tree/lsm.py` | **No** — flush only | **No** — flush only | **No** |
| `log-structured-hash-table/bitcask.py` | **No** — flush only | **Unknown** (code cut off) | **No** |
| `sstable-and-compaction/sstable.py` | **No** | **Unknown** (code cut off) | **No** |

## Detailed Findings

### write-ahead-log/wal.py — Properly Durable

This is the gold standard in the codebase. Every durability-critical path pairs `flush()` with `os.fsync()`:

- **Single append** (`_do_sync`, line 114-115): fsyncs in `"sync"` mode, batches in `"batch"` mode
- **Batch append** (`append_batch`, line 183-184): always force-fsyncs after the COMMIT record
- **Rotation** (`_rotate`, line 127-128): fsyncs the old file before opening a new one
- **Truncation** (`truncate`, line 208-209): fsyncs the rewritten file after removing old records
- **Checkpoint** (line 132-133): force-fsyncs

The WAL implementation understands that `flush()` alone only moves data from Python's userspace buffer to the OS page cache — it does **not** guarantee the data reaches stable storage. The consistent `flush()` → `os.fsync()` pattern ensures actual durability.

### b-tree-storage-engine/btree.py — Safe via WAL Protocol

The B-tree uses a two-phase approach:

- **`PageManager.write_page()`** (line ~85) and **`_write_meta()`** (line ~47): only call `self._f.flush()` — no fsync. This is intentional.
- **`PageManager.sync()`** (line ~111): calls `flush()` + `os.fsync()`.
- **`PageManager.close()`** (line ~115): calls `flush()` + `os.fsync()`.

The safety comes from the WAL:
1. `WAL.log_write()` (line ~140) writes page data to the WAL and fsyncs it
2. `WAL.commit()` (line ~147) calls `page_manager.sync()` (which fsyncs the data file), then truncates the WAL and fsyncs that too
3. `WAL.recover()` (line ~155) replays logged writes and fsyncs

This is correct: individual page writes don't need to be durable because the WAL can replay them after a crash. The `commit()` call is the durability barrier.

### log-structured-merge-tree/lsm.py — Not Durable at All

**This is the most concerning finding.** The LSM tree's internal WAL (class `WAL`, line 13) only calls `self._fd.flush()` at line 26 — **never** `os.fsync()`. Grep confirms zero `os.fsync` calls anywhere in this file.

- **Normal writes** (WAL append, line 26): `self._fd.flush()` only
- **SSTable flush** (`_flush`, line 303): writes to a new SSTable file — needs verification but no fsync appears in the grep results
- **Compaction** (`compact`, line 319): merges SSTables — code was cut off at line 200 in the observation, but the complete grep shows no fsync anywhere in the file

This means a crash at any point can lose data. The WAL is supposed to be the recovery mechanism, but since it never fsyncs, a power failure can lose WAL entries that were "flushed" to OS buffers but never reached disk. Compaction is doubly dangerous: if the process crashes mid-compaction, the new merged SSTable may be partially written (or entirely in page cache), old SSTables may already be deleted, and the WAL has already been truncated.

**Verdict: crash recovery here is purely lucky** — it works only because the OS eventually flushes dirty pages, and the tests never simulate actual power loss.

### hash-index-storage/bitcask.py — Conditionally Safe for Writes, Unknown for Compaction

Normal writes in `_write_record()` (line ~97-99):
```python
self.active_file.flush()
if self.sync_writes:
    os.fsync(self.active_file.fileno())
```

The `sync_writes` flag (defaulting to `True` in `__init__`, line 31) makes normal writes durable. But the `compact()` method starts at line ~213 and **was cut off in the observations** — we cannot confirm whether compaction output is fsynced.

### log-structured-hash-table/bitcask.py — Not Durable

`_write_record()` (line ~157) calls only `self._active_file.flush()` — no fsync, no conditional sync option. The compaction code was cut off in observations, but there are zero `os.fsync` calls in the grep results for this file.

### sstable-and-compaction/sstable.py — Not Durable

`SSTableWriter.finish()` (line ~89) closes the file after writing the index and footer, but never calls `flush()` or `os.fsync()`. The file close will flush Python buffers but does not guarantee data reaches disk. Compaction strategy code was cut off at line 200.

## The Core Problem

The dangerous pattern is:

1. Write new compacted file (data in OS page cache only)
2. Delete old segment files
3. Crash before OS flushes the new file to disk
4. On recovery: old files gone, new file empty or corrupt

Only `hash-index-storage/bitcask.py` with `sync_writes=True` protects normal writes. The `write-ahead-log/wal.py` and `b-tree-storage-engine/btree.py` are properly durable. The LSM tree, log-structured hash table, and SSTable implementations are all vulnerable.

## Observation Gaps

The compaction methods for three implementations (`hash-index-storage/bitcask.py:compact`, `log-structured-hash-table/bitcask.py:compact`, `sstable-and-compaction/sstable.py` compaction strategies) were **cut off at line 200**. A complete audit requires reading these methods to confirm whether they fsync compaction output before deleting old segments.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — Verify whether compacted output files are fsynced before old segments are deleted; this determines if the conditional durability from `sync_writes` extends to compaction
- [function] `log-structured-hash-table/bitcask.py:compact` — Check if compaction writes are any more durable than the flush-only normal writes; likely not, given no fsync appears in grep results
- [function] `log-structured-merge-tree/lsm.py:compact` — Read the full compaction method to confirm old SSTables are deleted without fsyncing the replacement, and whether WAL truncation creates a recovery gap
- [general] `directory-fsync-gap` — Even implementations that fsync file data may not fsync the parent directory after creating new files or renaming; on Linux/ext4, the directory entry itself can be lost without `os.fsync(os.open(dir, os.O_RDONLY))`
- [file] `sstable-and-compaction/sstable.py` — Read the compaction strategy classes (likely after line 200) to check whether size-tiered or leveled compaction adds any durability guarantees the writer lacks

## Beliefs

- `lsm-wal-never-fsyncs` — The LSM tree's internal WAL (`log-structured-merge-tree/lsm.py:WAL.append`) calls only `flush()` and never `os.fsync()`, making it unable to survive power loss
- `btree-durability-via-wal-commit` — The B-tree's page writes are intentionally not fsynced individually; durability comes from `WAL.log_write()` fsync + `WAL.commit()` calling `PageManager.sync()`, which is the correct two-phase pattern
- `wal-py-always-pairs-flush-fsync` — `write-ahead-log/wal.py` consistently pairs `flush()` + `os.fsync()` on every durability-critical path including rotation, truncation, and checkpointing
- `bitcask-sync-writes-flag-controls-fsync` — `hash-index-storage/bitcask.py` gates `os.fsync()` on the `sync_writes` constructor parameter (default `True`), but it is unknown whether compaction honors this flag
- `log-structured-hash-table-no-fsync` — `log-structured-hash-table/bitcask.py:_write_record` calls only `flush()` with no fsync and no option to enable it, making all writes vulnerable to power loss

