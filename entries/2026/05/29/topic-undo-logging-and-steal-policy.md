# Topic: Why ARIES needs undo at all: the STEAL buffer management policy lets dirty pages from uncommitted transactions be written to disk, which this simpler design avoids entirely

**Date:** 2026-05-29
**Time:** 08:25

# Why ARIES Needs Undo — and This Codebase Doesn't

## The Core Trade-off: STEAL vs. NO-STEAL

In database buffer management, the **STEAL** policy allows the buffer manager to evict dirty pages from uncommitted transactions to disk — "stealing" the page frame for another use. This is great for memory efficiency (you don't need to pin every page a long-running transaction touches), but it creates a problem: if the system crashes, the on-disk data file contains partial writes from transactions that never committed. Those writes must be **undone**.

ARIES handles this with before-images in the log: every modification records what the data looked like *before* the change, so recovery can reverse uncommitted work. This is roughly half of ARIES's complexity.

**This codebase sidesteps the entire problem by never writing uncommitted data to the persistent data store.**

## How Each Implementation Avoids STEAL

### The WAL: Redo-Only Log Records

Look at the operation types in `write-ahead-log/wal.py:10-13`:

```python
OP_PUT = 1
OP_DELETE = 2
OP_COMMIT = 3
OP_CHECKPOINT = 4
```

There is no `OP_UNDO`, no `OP_COMPENSATE`, no before-image field. Each `WALRecord` (`wal.py:17-22`) stores `key` and `value` — that's the **new** value only. There is no `old_value` field. The log is structurally incapable of undoing anything.

Recovery works by replaying forward: `append_batch` (`wal.py:148-163`) writes all operations atomically with a `COMMIT` marker, and `replay` (referenced in tests at `test_wal.py:22-25`) only returns records that form complete, committed groups. Uncommitted operations simply aren't replayed.

### MVCC: Uncommitted Data Lives Only in Memory

The snapshot isolation implementation in `snapshot-isolation/mvcc_database.py` is the clearest example. The entire version store is in-memory (`mvcc_database.py:53`):

```python
self._versions = {}  # key -> list[Version]
```

When a transaction writes, it appends a `Version` object to an in-memory list (`mvcc_database.py:144-150`). If the transaction aborts, its tx_id goes into `self._aborted` (`mvcc_database.py:55`), and the visibility check at `mvcc_database.py:80-81` filters it out:

```python
if created_by in self._aborted:
    return False
```

There is no disk page to undo because uncommitted data never reached disk. This is a **NO-STEAL** policy by construction — the "buffer pool" is the entire in-memory dict, and it's never partially flushed.

### SSI: Buffered Writes, Not Buffered Pages

The SSI implementation (`write-skew-detection/ssi_database.py:12`) keeps transaction writes in a per-transaction buffer:

```python
self._writes = {}  # key -> value (buffered writes)
```

An abort (`ssi_database.py:298-300`) just marks the transaction status — the buffered writes are discarded with the transaction object. The underlying data store never saw them.

### B-Tree WAL: Atomic Page Writes, Not Transaction Undo

The B-tree's WAL (`b-tree-storage-engine/btree.py:119-170`) serves a different purpose than ARIES: it provides **atomic multi-page writes** (e.g., during a node split), not transaction rollback. The `recover` method (`btree.py:150-168`) replays page writes forward and then truncates the log. There's no concept of undoing a page write — either the whole operation committed (pages synced, WAL truncated via `commit` at `btree.py:139-145`) or it didn't (WAL replayed on restart).

## What ARIES Gains from STEAL (That This Design Gives Up)

The NO-STEAL approach is simpler, but it has a cost: **all dirty data from active transactions must fit in memory.** A long-running transaction that modifies millions of rows must keep all those modifications buffered. ARIES's STEAL policy lets the buffer manager evict those pages to disk under memory pressure, at the cost of needing undo logging for crash recovery.

For these reference implementations — which are teaching tools with in-memory data stores — that trade-off is irrelevant. For a production database handling terabyte-scale transactions, it's the reason ARIES exists.

## Topics to Explore

- [function] `snapshot-isolation/mvcc_database.py:_is_visible` — The visibility rules that make abort "free" — understanding why filtering by `_aborted` is sufficient without disk undo
- [function] `b-tree-storage-engine/btree.py:recover` — Compare this redo-only page recovery with ARIES's three-phase recovery (analysis, redo, undo)
- [function] `write-ahead-log/wal.py:append_batch` — How atomic commit markers eliminate the need for undo by making uncommitted batches invisible to replay
- [general] `aries-three-pass-recovery` — ARIES's analysis/redo/undo phases and why the undo phase disappears under NO-STEAL
- [file] `write-skew-detection/ssi_database.py` — How per-transaction write buffering interacts with abort semantics when no data reaches disk

## Beliefs

- `wal-has-no-before-images` — WALRecord contains only the new value (key, value) with no old_value field, making undo structurally impossible from the log alone
- `mvcc-uncommitted-data-memory-only` — In MVCCDatabase, uncommitted transaction writes exist only as in-memory Version objects in `_versions` and never reach a persistent data store
- `abort-is-status-change-not-disk-rollback` — Aborting a transaction in both MVCC and SSI implementations sets a status flag; no disk writes are reversed
- `btree-wal-is-redo-only` — The B-tree WAL's `recover` method replays logged page writes forward and truncates; it has no mechanism to reverse a page write

