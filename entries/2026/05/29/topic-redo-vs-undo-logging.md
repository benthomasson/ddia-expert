# Topic: This WAL uses redo-only logging (after-images); contrast with undo logging (before-images) and ARIES-style redo/undo to understand the tradeoffs

**Date:** 2026-05-29
**Time:** 10:27

# Redo-Only Logging in This WAL Implementation

## What "redo-only" means here

Look at `write-ahead-log/wal.py:131-140`, the `append` method. When a PUT operation is logged, the record contains `(seq_num, op_type, key, value)` — the **new value** being written. There is no field for the old value that the key previously held. The same pattern holds in `append_batch` (lines 142-158): each operation in the batch records what *will* be written, not what *was* there before.

This is the defining characteristic of **redo-only (after-image) logging**: the WAL stores enough information to *replay* committed operations forward from some known-good state, but not enough to *reverse* them.

You can see this concretely in the `WALRecord` dataclass at line 22:

```python
@dataclass
class WALRecord:
    seq_num: int
    op_type: str
    key: str
    value: str      # ← the new value (after-image), never the old value
    checksum: int
```

Recovery works by replaying these after-images. The `replay` method (not shown in the excerpt but referenced in `test_wal.py:35`) reads records forward and re-applies them. The LSM tree's recovery at `log-structured-merge-tree/lsm.py:230-231` shows the same pattern:

```python
def _replay_wal(self):
    for key, value in self._wal.replay():
```

It blindly re-inserts every key-value pair from the WAL into the memtable. No old values are consulted or needed.

## Contrast: Undo logging (before-images)

An **undo log** records the **old value** (before-image) for each modified key. Its purpose is the opposite: it lets you *reverse* uncommitted changes after a crash, restoring the database to its pre-transaction state.

The tradeoffs flip:

| Property | Redo-only (this WAL) | Undo-only |
|---|---|---|
| **What's logged** | New value (after-image) | Old value (before-image) |
| **Recovery direction** | Replay forward from checkpoint | Roll back uncommitted changes |
| **When data hits disk** | After commit (can buffer) | Before commit (must flush dirty pages first) |
| **Checkpoint requirement** | Must checkpoint periodically to bound replay | Less critical — committed data already on disk |
| **Abort cost** | Cheap (just discard WAL entries) | Must write compensation records |

The critical operational difference: undo logging forces a **write-ahead constraint on data pages** — dirty pages must reach disk *before* the commit record, so that a crash never leaves committed-but-unwritten data. This WAL takes the opposite approach. Look at `append_batch` (line 156): the COMMIT record is written and fsynced *to the WAL*, but the actual data pages (memtable, SSTables) can be flushed lazily. The data might only exist in the WAL until the next checkpoint or compaction.

You can see this play out in `test_wal.py:29-41` (`test_crash_recovery`): a second `WriteAheadLog` instance replays the WAL and recovers the data purely from after-images, without needing any "before" state.

## Contrast: ARIES-style redo/undo

ARIES (Algorithm for Recovery and Isolation Exploiting Semantics) combines both approaches. It logs **both** before-images and after-images, plus an LSN (Log Sequence Number) on every data page. This enables:

1. **Redo pass**: replay all operations (committed or not) to bring pages to their crash-time state
2. **Undo pass**: roll back any transactions that were active at crash time
3. **Fine-grained page-level recovery**: the page LSN tells recovery exactly which log records have already been applied, avoiding duplicate redo

This WAL's `seq_num` (line 22) is conceptually similar to an LSN, and checkpoint records (`OP_CHECKPOINT` at line 14, written at line 160) serve a similar purpose to ARIES's checkpoint — bounding how far back replay must go. But the similarity ends there. This implementation lacks:

- **Before-images**: no ability to undo partial transactions
- **Page LSNs**: no mechanism to detect whether a given operation was already applied to the data store
- **Active transaction table**: no tracking of in-flight transactions during checkpoint

This is fine for the use case. The MVCC database in `snapshot-isolation/mvcc_database.py` handles transaction isolation at a higher level using version chains (lines 11-16), not WAL-level undo. And the LSM/Bitcask engines are append-only — there's nothing to "undo" because old data is never overwritten in place.

## Why redo-only is the right choice here

These storage engines (Bitcask at `hash-index-storage/bitcask.py`, LSM at `log-structured-merge-tree/lsm.py`) are **append-only**. Writes never modify existing data on disk — they always append new records. This eliminates the core problem undo logging solves (restoring overwritten pages). The WAL just needs to recover the memtable contents that hadn't been flushed to SSTables or data files yet.

The cost is that recovery time is proportional to the number of records since the last checkpoint/truncation. The `truncate` method (`wal.py:170`) and checkpoint records exist specifically to bound this cost.

## Topics to Explore

- [function] `write-ahead-log/wal.py:append_batch` — Shows how atomic multi-operation commits work without undo capability: the COMMIT record is the atomicity boundary, and a crash mid-batch means the incomplete batch is simply not replayed
- [function] `log-structured-merge-tree/lsm.py:_replay_wal` — The consumer side of redo-only recovery: blindly re-applies WAL entries into the memtable with no idempotency checks or duplicate detection
- [file] `snapshot-isolation/mvcc_database.py` — Implements transaction isolation via version chains rather than WAL-level undo, showing how MVCC separates the isolation concern from the durability concern
- [general] `checkpoint-truncation-interaction` — How `checkpoint()` and `truncate()` work together to bound recovery time — the key operational cost of redo-only logging
- [function] `hash-index-storage/bitcask.py:_rebuild_index` — An alternative recovery strategy: instead of WAL replay, Bitcask scans its append-only data files directly, showing that redo logging is unnecessary when the data format itself is a log

## Beliefs

- `wal-records-contain-only-after-images` — WALRecord stores (key, new_value) with no field for the previous value; the log cannot reverse an operation, only replay it
- `wal-batch-atomicity-via-commit-marker` — `append_batch` writes all operations plus a trailing COMMIT record in a single fsync; replay logic uses the COMMIT marker to determine whether a batch completed, discarding partial batches without needing undo
- `wal-checkpoint-bounds-replay` — The CHECKPOINT record (op_type 4) marks a point after which all prior data is durable elsewhere; `replay(after_seq=cp_seq)` skips records at or before the checkpoint, bounding recovery time
- `wal-seq-num-is-monotonic-not-page-lsn` — `seq_num` increases monotonically across all records but is not stamped onto data pages; there is no mechanism to detect whether a replayed operation was already applied, making replay non-idempotent against the underlying store

