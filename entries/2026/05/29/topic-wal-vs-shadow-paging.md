# Topic: Compare WAL-based crash recovery with shadow paging (copy-on-write) as used in LMDB and SQLite's legacy rollback journal

**Date:** 2026-05-29
**Time:** 10:57

# WAL-Based Crash Recovery vs. Shadow Paging

## What the codebase shows — and what it doesn't

This repo implements WAL-based crash recovery in three different storage engines. **It does not implement shadow paging.** The grep for `shadow`, `copy.on.write`, `cow`, and `rollback` found zero relevant hits. So this explanation covers WAL in depth from the code, and contrasts with shadow paging conceptually based on how LMDB and SQLite's rollback journal work.

---

## WAL-Based Recovery: The Core Idea

Write-ahead logging enforces a single rule: **before any data page is modified on disk, the intended change is first written and fsynced to a separate log file.** If the process crashes mid-write, the log contains enough information to either redo or undo the incomplete operation.

This repo has three WAL implementations, each with a different scope:

### 1. Standalone WAL (`write-ahead-log/wal.py`)

This is the most feature-complete. It logs logical operations (PUT, DELETE) as self-describing records with CRC checksums.

The write path at `wal.py:143-152` shows the protocol:

```python
def append(self, op_type, key, value=""):
    self._seq_num += 1
    data = _encode_record(seq, OP_BYTES[op_type], key.encode(), value.encode())
    self._fd.write(data)
    self._do_sync()
```

Each record (`wal.py:28-34`) carries a sequence number, operation type, key, value, and CRC32 checksum. The sequence number is the backbone of recovery — on restart, `_recover_seq_num()` (`wal.py:87-98`) scans all WAL files to find the highest valid sequence, so new writes never collide with old ones.

Corruption detection is straightforward (`wal.py:53-57`): recompute CRC over the operation byte + key + value, compare against the stored CRC. On mismatch, replay stops — no attempt to skip ahead. The test at `test_wal.py:48-57` confirms this: corrupting the last 5 bytes of a WAL file causes the second record to be lost while the first survives.

**Batch atomicity** (`wal.py:155-170`) is achieved by buffering all operations into a single byte array, appending a COMMIT record, then writing and fsyncing the entire buffer at once. During replay, uncommitted operations (those without a trailing COMMIT) are simply skipped.

### 2. Physical-Page WAL (`b-tree-storage-engine/btree.py:130-185`)

This WAL operates at the page level rather than the logical level. It logs **entire page images** before they're written to the data file:

```python
def log_write(self, page_num, page_data):
    self._seq += 1
    header = struct.pack(self.ENTRY_HEADER, self._seq, page_num, len(page_data))
    checksum = struct.pack('>I', self._checksum(page_data))
    self._f.write(header + page_data + checksum)
    self._f.flush()
    os.fsync(self._f.fileno())
```

The `commit()` method (`btree.py:148-153`) is the critical two-phase step:
1. Fsync the data file (ensuring all replayed pages are durable)
2. Truncate the WAL to zero and fsync it

If a crash occurs between steps 1 and 2, recovery simply re-applies the WAL entries — they're idempotent because they write full page images. The `recover()` method (`btree.py:155-178`) reads each entry, validates its checksum, and writes the page back to the data file.

### 3. Minimal WAL (`log-structured-merge-tree/lsm.py:13-63`)

The LSM tree's WAL is intentionally bare — no checksums, no sequence numbers, no commit markers. It appends length-prefixed key-value pairs and replays them into the memtable on recovery. This is safe because the LSM tree treats the memtable as transient anyway; the WAL just prevents data loss between flushes.

---

## Shadow Paging: The Alternative (Not Implemented)

Shadow paging takes the opposite approach: **instead of logging what you intend to change, you write changes to new pages and atomically swap a root pointer.**

**How LMDB does it:** LMDB maintains two copies of the root (meta) page. A write transaction copies every B-tree page it modifies to a new location (copy-on-write), building a new tree that shares unmodified pages with the old one. When done, it writes the new root to the *inactive* meta page and fsyncs. The root pointer swap is atomic because a single meta page write either completes or doesn't — there's no intermediate state. Recovery is trivial: read both meta pages, use the one with the higher transaction ID that has a valid checksum.

**How SQLite's rollback journal does it:** Before modifying a page in the database file, SQLite copies the *original* page to a rollback journal. If the transaction commits, the journal is deleted. If the process crashes, the journal contains the pre-modification pages needed to restore the database to its last consistent state. This is shadow paging in reverse — the "shadow" is the old state, not the new.

### Why the distinction matters

| Property | WAL (this codebase) | Shadow Paging (LMDB-style) |
|----------|---------------------|---------------------------|
| **Write amplification** | Log entry + eventual data write | Copy every modified page + ancestors to root |
| **Read path during recovery** | Must replay log sequentially | Just read the correct root — instant |
| **Concurrent readers** | Need coordination (or MVCC) | Readers see a consistent snapshot for free (old root) |
| **Sequential I/O** | WAL appends are sequential; excellent for HDDs | Page copies scatter across the file |
| **Space overhead** | Log grows until truncated/checkpointed | Must keep free pages for shadows; fragmentation risk |
| **Recovery time** | Proportional to log length since last checkpoint | Constant — just pick the valid root |

The B-tree WAL in this codebase (`btree.py:130-185`) is the closest analog to compare against LMDB. Both operate on fixed-size pages and both use checksums for integrity. But the B-tree WAL writes pages to a log first and then copies them to the data file, while LMDB writes modified pages to *new locations in the data file itself* and swaps the root pointer. LMDB never overwrites live data, which means it never needs a recovery phase — but it does need a free-page manager to reclaim shadow pages after all readers using the old root have finished.

---

## What's Missing from This Codebase

To do a true side-by-side comparison, the repo would need:
- A copy-on-write B-tree (LMDB-style) to show shadow paging concretely
- A rollback journal implementation (SQLite legacy mode) to show the "undo log" variant
- Benchmarks comparing recovery time, write amplification, and concurrent read behavior

The existing `PageManager.free_page()` method (`btree.py:97-102`) hints at the free-list machinery that shadow paging requires, but it's used for B-tree node reuse, not copy-on-write.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:WAL.recover` — Trace the full recovery path: how page-level redo replay works and why idempotency makes it safe
- [function] `b-tree-storage-engine/btree.py:WAL.commit` — The two-phase commit protocol between data file fsync and WAL truncation; what happens if a crash occurs between them
- [function] `write-ahead-log/wal.py:append_batch` — How batch atomicity is achieved with a single buffered write + COMMIT record, and why this doesn't need two-phase commit
- [general] `lmdb-copy-on-write-btree` — Study LMDB's source (particularly `mdb.c`) to see how copy-on-write page allocation, dual meta pages, and reader snapshots interact — the missing half of this comparison
- [function] `b-tree-storage-engine/btree.py:PageManager.free_page` — The free-list mechanism that a shadow-paging implementation would depend on heavily for reclaiming obsolete page copies

## Beliefs

- `wal-stops-replay-on-crc-failure` — The standalone WAL (`wal.py:53-57`) halts replay at the first CRC mismatch rather than attempting to skip corrupted records and continue; all subsequent records are lost
- `btree-wal-logs-full-page-images` — The B-tree WAL (`btree.py:136-146`) logs complete page contents (not deltas), making recovery replay idempotent — re-applying the same entry produces the same result
- `btree-wal-commit-requires-data-fsync-before-truncate` — `WAL.commit()` (`btree.py:148-153`) fsyncs the data file before truncating the WAL; reversing this order would risk data loss on crash
- `lsm-wal-has-no-checksums` — The LSM tree WAL (`lsm.py:13-63`) stores no checksums or sequence numbers, relying on length-prefix framing alone for integrity; a partial write mid-record will silently truncate replay
- `no-shadow-paging-implementation-exists` — The codebase contains no copy-on-write or shadow paging implementation; all three storage engines use write-ahead logging exclusively for crash recovery

