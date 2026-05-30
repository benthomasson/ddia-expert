# Topic: How production systems (PostgreSQL, SQLite, InnoDB) encode multi-step structural operations in their WAL so recovery can complete or roll back partial work

**Date:** 2026-05-29
**Time:** 10:47

# How WAL Encodes Multi-Step Structural Operations

## What the Codebase Shows

This codebase implements three distinct WAL strategies, each at a different level of sophistication. Together they illustrate the core tension: a single logical operation (like a B-tree split) touches multiple pages, and the WAL must ensure recovery either completes all of them or none.

### Strategy 1: Logical Record WAL (write-ahead-log/wal.py)

The standalone WAL in `write-ahead-log/wal.py:12-14` defines four operation types:

```python
OP_PUT = 1
OP_DELETE = 2
OP_COMMIT = 3
OP_CHECKPOINT = 4
```

Multi-step atomicity is handled by `append_batch` (`wal.py:153-168`), which buffers all operations into a single `bytearray`, appends a `COMMIT` record, then writes the entire buffer in one `_fd.write()` call followed by a forced fsync. The COMMIT record acts as the atomicity boundary — during replay, any operations not followed by a COMMIT are discarded as incomplete.

This is **logical logging**: the WAL records the operation (put key X with value Y), not the physical page changes. Recovery replays the logical operations to reconstruct state.

### Strategy 2: Physical Page-Image WAL (b-tree-storage-engine/btree.py)

The B-tree's WAL (`btree.py:117-178`) takes the opposite approach — **physical logging**. Each entry records a full page image:

```python
# Entry format: seq(4B) + page_num(4B) + data_len(4B) + data + checksum(4B)
```

The `log_write` method (`btree.py:130-137`) writes the entire new page contents before they're applied to the data file. The `commit` method (`btree.py:139-144`) syncs the data file, then truncates the WAL to zero. Recovery (`btree.py:146-168`) replays every logged page image back to the data file.

For a B-tree split — which modifies the original leaf, creates a new leaf, and updates the parent — the caller would call `log_write` for each affected page. But here's the critical gap: **there is no commit marker within the WAL itself**. The WAL is truncated only after all pages are synced to the data file. If a crash occurs mid-replay, recovery simply replays everything again (page writes are idempotent).

### Strategy 3: Append-Only WAL (log-structured-merge-tree/lsm.py)

The LSM WAL (`lsm.py:13-65`) is the simplest — it's just a sequential append of key-value pairs with no transaction boundaries at all. Every entry is independently valid. This works because the LSM tree's structure (memtable + immutable SSTables) means the WAL only protects the mutable memtable; structural operations like compaction create new SSTables atomically via file rename.

## What's Missing: How Production Systems Actually Do This

**The codebase doesn't implement multi-step structural operation tracking**, and this is where production systems diverge significantly. Here's what each does:

### PostgreSQL: Physiological Logging + Incomplete-Action Tracking

PostgreSQL uses **physiological logging** — a hybrid where the WAL identifies a page by physical address but describes the change logically (e.g., "insert tuple at offset 3 on page 42"). For a B-tree split, PostgreSQL writes multiple WAL records tagged with the same transaction ID, including a special "incomplete action" flag on the first record of the split sequence. Recovery tracks these flags: if it sees the start of a split but not the completion, it finishes the split during redo. The key insight is that structural operations like splits are **not transactional** — they must complete regardless of whether the user's transaction commits. PostgreSQL handles this by treating them as separate "WAL-logged actions" that recovery always drives to completion.

### InnoDB: Mini-Transactions (mtr)

InnoDB groups related page modifications into **mini-transactions**. An mtr is an atomic unit of redo log records — either all records in the mtr are applied during recovery, or none are. A B-tree split generates one mtr containing redo records for: (1) the original page modification, (2) the new page allocation and initialization, (3) the parent page update. The mtr is written to the redo log with a special end marker. InnoDB also maintains separate **undo logs** for rolling back user transactions, but structural changes (splits/merges) use only redo and are never rolled back.

### SQLite: Rollback Journal / WAL Mode

SQLite takes yet another approach. In rollback journal mode, it copies the **original page images** (before-images) to the journal before modifying the database file. To roll back, it restores the original pages. In WAL mode (closer to what `btree.py` implements), new page images go into the WAL, and the original database is untouched until checkpoint. Multi-page operations are bracketed by frame headers that include a commit flag — only frames up to and including a commit frame are visible to readers.

### The Common Pattern

All three production systems share a principle absent from this codebase: **structural operations (splits, merges, rebalancing) are encoded as atomic groups in the WAL that recovery always drives forward, independent of user transaction commit/abort**. The B-tree WAL in `btree.py` approximates this with its "replay everything then truncate" approach, but it doesn't handle the case where a crash occurs between writing some pages to the data file and truncating the WAL — it relies on idempotent page writes rather than explicit completion tracking.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:WAL.recover` — The replay-and-truncate strategy is the closest this codebase gets to production WAL recovery; trace what happens if a crash occurs mid-recovery
- [function] `write-ahead-log/wal.py:WriteAheadLog.append_batch` — Study how the COMMIT record creates an atomicity boundary and what happens to uncommitted operations during replay
- [general] `physiological-logging` — PostgreSQL's hybrid approach (physical page address + logical change description) is the industry standard; compare with the full-page-image approach in btree.py
- [general] `innodb-mini-transactions` — InnoDB's mtr concept is the cleanest model for understanding how multi-page structural changes become atomic in the WAL
- [function] `log-structured-merge-tree/lsm.py:WAL.replay` — Compare why an LSM tree can get away with no commit markers (structural changes happen via atomic file operations, not in-place page mutations)

## Beliefs

- `btree-wal-logs-full-page-images` — The B-tree WAL in `btree.py:130-137` logs complete page contents (physical logging), not logical operations, making replay idempotent but space-expensive
- `wal-commit-record-is-atomicity-boundary` — In `wal.py:153-168`, `append_batch` writes all operations plus a COMMIT record in a single buffer+fsync; operations without a trailing COMMIT are invisible to replay
- `btree-wal-has-no-multi-step-grouping` — The B-tree WAL lacks transaction or mini-transaction markers, relying entirely on truncate-after-sync rather than explicit completion tracking for multi-page operations like splits
- `lsm-wal-has-no-commit-markers` — The LSM WAL (`lsm.py:13-65`) appends bare key-value pairs with no commit boundaries, safe only because structural changes (compaction, SSTable creation) use atomic file operations outside the WAL

