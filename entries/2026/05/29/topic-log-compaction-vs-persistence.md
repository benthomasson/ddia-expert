# Topic: The gap between in-memory truncation and on-disk state: how a real system would handle WAL segment files and checkpointing

**Date:** 2026-05-29
**Time:** 08:31

# The Gap Between In-Memory Truncation and On-Disk State

## What the Implementation Does

The WAL in `write-ahead-log/wal.py` has a `truncate` method (line 179) that performs **record-level filtering within segment files**. It closes the current file descriptor, iterates through every `.wal` file, reads each record, keeps only those with `seq_num > up_to_seq`, and either rewrites the file with the survivors or deletes it entirely if nothing survives. This is a correct but expensive approach — it's essentially a full rewrite of every segment file on every truncation call.

The LSM tree in `log-structured-merge-tree/lsm.py` takes an even simpler approach: its `WAL.truncate()` (line 56) just opens the file in write mode (`"wb"`), which zeros it out completely. The LSM calls this at line 314 after flushing the memtable to an SSTable — a clean "the WAL served its purpose, nuke it" pattern.

## Where Real Systems Diverge

### 1. Segment-level truncation, not record-level

The WAL implementation already has **segment rotation** (`_rotate` at line 112, `_maybe_rotate` at line 131) — it creates new numbered files like `000001.wal`, `000002.wal` when a segment exceeds `max_file_size`. But `truncate` doesn't exploit this. It opens every file and filters record-by-record.

A production system (PostgreSQL, RocksDB, etcd) would instead track which segments are fully checkpointed and **delete entire segment files**. The reasoning: if your checkpoint is at sequence 50,000 and segment `000003.wal` contains sequences 30,001–45,000, you can `os.unlink` the whole file. No reading, no rewriting. You only need to handle the boundary segment — the one that straddles the checkpoint point — and even that is often left intact until it's entirely superseded.

### 2. The checkpoint record is disconnected from actual checkpointing

The `checkpoint` method (line 169) writes an `OP_CHECKPOINT` record into the WAL. But it doesn't coordinate with any external state. In a real system, a checkpoint means:

1. **Flush dirty pages** (or the memtable) to durable storage
2. **Record the checkpoint LSN** — the sequence number up to which all data is safely on disk
3. **Then** truncate WAL segments that precede the checkpoint LSN

The LSM tree gets closest to this pattern: it flushes the memtable to an SSTable (`_flush` at line 303), then calls `self._wal.truncate()` (line 314). But notice the LSM's WAL has no checkpoint record at all — it just wipes the file. The standalone WAL has checkpoint records but no flush-to-stable-storage step. Neither implementation connects both halves.

### 3. Missing: crash-safe truncation

Look at what `truncate` does in `wal.py` starting at line 179:

```python
self._fd.flush()
os.fsync(self._fd.fileno())
self._fd.close()
self._fd = None

for path in self._wal_files():
    # read, filter, rewrite or delete
```

If the process crashes mid-truncation — say after rewriting `000002.wal` but before rewriting `000003.wal` — you end up with a mix of truncated and untruncated files. The data isn't lost (replaying would just re-apply already-applied operations, which is fine if the operations are idempotent), but the implementation doesn't explicitly handle this.

Real systems avoid this by never rewriting WAL files. They either:
- Delete entire segments (atomic at the filesystem level)
- Maintain a separate metadata file recording the checkpoint LSN, and let recovery skip records below it

### 4. No `fsync` on the data file before truncating the WAL

The LSM tree's `_flush` method (line 303) writes an SSTable but never calls `os.fsync` on the SSTable file. It then truncates the WAL. If the OS crashes after the WAL truncate but before the SSTable data reaches disk, the data is gone from both. The standalone WAL's `truncate` at least syncs its own file before closing, but there's no protocol ensuring the *destination* of the checkpoint is durable before the *source* is discarded.

This is the single most critical gap: **the WAL must not be truncated until the data it protects is confirmed durable via `fsync` on the target.**

### 5. Recovery doesn't use checkpoint records

In `test_wal.py` line 24, the test calls `wal.replay(after_seq=cp_seq)` to skip past the checkpoint. But this is caller-driven — the caller has to remember the checkpoint sequence number. A production system would scan backward from the end of the WAL to find the last `OP_CHECKPOINT` record and start replay from there automatically, eliminating the need for the caller to track this externally.

## Summary

The implementation has the right **vocabulary** — segments, rotation, checkpoint records, truncation, CRC integrity checks — but the pieces aren't wired together the way a production system demands. The critical missing invariant is: *truncation is a consequence of a confirmed-durable checkpoint, and segment files are the unit of truncation*. Without that, you get correct behavior on the happy path but data loss or unnecessary I/O on the edges.

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:_flush` — The closest thing to a real checkpoint: memtable-to-SSTable flush followed by WAL truncation, but missing fsync on the SSTable
- [function] `write-ahead-log/wal.py:_rotate` — Segment rotation creates the file boundaries that production truncation would exploit, but truncation ignores them
- [file] `unbundled-database/unbundled_database.py` — Has a `truncate_before(lsn)` API (line 58) that takes an LSN parameter, suggesting a more realistic checkpoint-driven truncation model
- [general] `fsync-ordering-guarantees` — How production databases ensure the write-order dependency between flushing data pages and truncating the WAL (the "write barrier" problem)
- [function] `log-structured-hash-table/bitcask.py:_find_existing_segments` — Bitcask's segment management shows an alternative approach where compaction (not checkpointing) drives old segment deletion

## Beliefs

- `wal-truncate-rewrites-files` — `WriteAheadLog.truncate()` reads and rewrites every segment file record-by-record rather than deleting whole segment files, making it O(total records) instead of O(segments)
- `lsm-wal-truncate-is-full-wipe` — The LSM tree's `WAL.truncate()` unconditionally zeros the single WAL file regardless of what has been checkpointed, coupling WAL lifetime to memtable flush
- `checkpoint-record-not-used-in-recovery` — `OP_CHECKPOINT` records are written to the WAL but recovery (`_recover_seq_num`) does not search for them; the caller must supply the checkpoint sequence number externally via `replay(after_seq=)`
- `no-fsync-before-wal-truncation` — Neither the standalone WAL nor the LSM tree ensures the checkpoint target (SSTable or equivalent) is fsynced before truncating the WAL, creating a crash window for data loss

