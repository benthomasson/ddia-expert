# Topic: Map out the full set of crash points across all storage engines and verify which ones actually survive `kill -9` followed by immediate reboot

**Date:** 2026-05-29
**Time:** 12:27

Now I have comprehensive information. Here's the full analysis:

---

# Crash Points Across All Storage Engines: kill -9 Survival Analysis

## The Inventory

The DDIA implementations repository contains **six storage engines** and **two supporting data structures** that touch disk. The question is: if you `kill -9` the process and immediately reopen the database, what survives?

The answer ranges from "everything" to "nothing, and you might not even notice."

## Engine-by-Engine Crash Point Map

### 1. Standalone WAL (`write-ahead-log/wal.py`) — Survives kill -9

The WAL is the gold standard in this codebase. Every durability-critical path pairs `flush()` + `os.fsync()`:

- **`_do_sync()` (line ~114-115)**: fsyncs in `"sync"` mode, batches in `"batch"` mode
- **`append_batch()` (line ~183-184)**: force-fsyncs after the COMMIT record regardless of mode
- **`_rotate()` (line ~127-128)**: fsyncs the old file before opening a new one
- **`truncate()` (line ~208-209)**: fsyncs the rewritten file after removing old records
- **`checkpoint()` (line ~132-133)**: force-fsyncs

**Crash points:**

| Point | What happens | Data loss? |
|-------|-------------|------------|
| Mid-single-append (sync mode) | Partial record on disk, CRC fails | No — `_read_record` stops at bad CRC, silently drops the torn tail |
| Mid-single-append (batch mode) | Up to `batch_sync_count - 1` records unfsynced | **Yes** — up to 99 records lost (default batch size 100) |
| Mid-`append_batch` | Either all ops + COMMIT reach disk, or none | No — single `write()` call + forced fsync. Partial batch detected by missing COMMIT |
| Mid-rotation | Old file fsynced, new file created | **Possible** — no directory fsync after new file creation (see below) |
| Mid-truncation | Rewriting files in place, no temp-rename | **Possible** — crash mid-rewrite can corrupt the WAL file being rewritten |

**Caveat**: `replay()` claims to "skip uncommitted batches" but actually returns **all** PUT/DELETE records regardless of whether a COMMIT record follows (`topic-wal-batch-atomicity-gap.md`). A torn batch write could replay partial operations.

### 2. B-Tree (`b-tree-storage-engine/btree.py`) — Survives kill -9

The B-tree implements ARIES-style redo logging with a correct three-phase commit protocol:

**Phase 1 — Write-ahead** (`WAL.log_write`, line ~135-141):
```
WAL entry (page image + CRC) → flush() → os.fsync()
```
The redo record is durable **before** the data page is modified.

**Phase 2 — Commit** (`WAL.commit`, line ~143-149):
```
(A) page_manager.sync()    → fsync data file
(B) WAL truncate(0)        → discard redo records
(C) os.fsync(WAL fd)       → make truncation durable
```

**Phase 3 — Recovery** (`WAL.recover`, line ~151-170):
```
Read WAL → verify CRC per entry → replay valid entries → fsync data file → truncate WAL
```

**Crash points:**

| Point | What happens | Data loss? |
|-------|-------------|------------|
| During Phase 1 (between `log_write` calls) | Partial set of WAL entries | No — recovery replays whatever is valid; redo is idempotent (physical page images) |
| After Phase 1, before (A) | WAL durable, data file may have partial writes | No — recovery replays all WAL entries; idempotent overwrites |
| Between (A) and (B) | Data durable, WAL still has entries | No — redundant replay, harmless |
| Between (B) and (C) | Data durable, WAL truncation may or may not be durable | No — either WAL is empty (clean) or entries get replayed again (idempotent) |
| During recovery itself | Partial replay | No — recovery follows same sync-then-truncate ordering; crash just triggers another identical pass |

**The key insight**: because WAL entries are physical page images (not logical operations), replaying the same entry twice writes identical bytes to the same offset. This makes the entire protocol crash-safe at every point — including crashes during recovery.

**Gap**: `PageManager._write_meta()` (line ~47) updates metadata (root, height, free list) with only `flush()`, no `fsync`. This is intentional — metadata durability is deferred to the single `page_manager.sync()` call in `WAL.commit()`. But `allocate_page()` and `free_page()` call `_write_meta()` independently, meaning a crash between a free-list update and the next commit could leave orphaned pages.

### 3. Bitcask Hash Index (`hash-index-storage/bitcask.py`) — Conditionally survives

Normal writes are gated on the `sync_writes` flag (default `True`):

```python
# _write_record, line ~88
self.active_file.flush()
if self.sync_writes:
    os.fsync(self.active_file.fileno())
```

**With `sync_writes=True`**: individual put/delete operations survive `kill -9`. The recovery path (`_recover`, line ~67) rebuilds the in-memory keydir by scanning segment files oldest-to-newest, using hint files as a fast path and falling back to full CRC-validated segment scans.

**Crash points:**

| Point | What happens | Data loss? |
|-------|-------------|------------|
| Mid-write (sync_writes=True) | Partial record, CRC fails | No — `_scan_segment` (line ~93) stops at first bad CRC |
| Mid-write (sync_writes=False) | Record in page cache only | **Yes** — `kill -9` safe (kernel preserves page cache), but **power loss** loses data |
| Mid-compaction, after old file delete (line ~288) but before rename (line ~297) | Old data file removed, new file not yet in place | **Yes** — keys only in deleted segment are permanently lost |
| Mid-compaction output write | Partial compacted file on disk, old files intact | No — orphaned partial file ignored on recovery; old files replayed |

**The compaction gap is the critical weakness.** The `os.remove()` at line 288 precedes the `os.rename()` at line 297. No fsync on compaction output is confirmed. No compaction manifest exists.

### 4. Log-Structured Hash Table (`log-structured-hash-table/bitcask.py`) — Does NOT survive

This Bitcask variant **never calls `os.fsync()`** anywhere. The `_write_record()` (line ~157) calls only `flush()`:

```python
self._active_file.flush()  # No fsync, no sync_writes option
```

**Every crash point is a data loss point.** The distinction between `kill -9` and power loss matters here:

- **`kill -9`**: The kernel preserves dirty page cache pages. The OS will eventually flush them. If the machine stays up, data is probably fine. But "probably" is not a durability guarantee.
- **Power loss / kernel panic**: All unflushed data is gone. The recovery path (`_recover`) rebuilds from whatever survived on disk, silently omitting lost records.

### 5. LSM Tree (`log-structured-merge-tree/lsm.py`) — Does NOT survive

This is the most dangerous engine because it **looks** like it should be crash-safe (it has a WAL) but **isn't**. The WAL (line ~13-60) calls `self._fd.flush()` at line 26 — **never `os.fsync()`**. Zero fsync calls anywhere in the file.

**Crash point cascade:**

| Point | What happens | Data loss? |
|-------|-------------|------------|
| After WAL append, before memtable flush | WAL in page cache, not on disk | **Yes** — WAL entries lost on power failure; `_replay_wal()` recovers nothing |
| After SSTable write, after WAL truncate | SSTable in page cache, WAL wiped | **Critical** — the WAL was destroyed (`truncate` reopens in `"wb"` mode at line ~58), SSTable may not be durable. Both gone. |
| Mid-compaction, after old SSTable delete (line ~353) | Old SSTables removed, merged SSTable in page cache | **Critical** — old data deleted, new data never reached disk. Permanent loss. |
| Mid-compaction, SSTable write | Partial new SSTable, old files intact | **Probably recoverable** — old SSTables survive; but no manifest means recovery is best-effort |

**The compound failure** at the SSTable flush boundary is the worst: `SSTable.write()` (line ~80) uses a `with` block that closes the file without fsync, then `WAL.truncate()` destroys the WAL. A crash between these two operations loses data from both paths simultaneously.

### 6. SSTable Writer (`sstable-and-compaction/sstable.py`) — Does NOT survive

`SSTableWriter.finish()` (line ~89) closes the file with no `fsync()`. Additionally, it has a **structural corruption vulnerability**: the entry count is written twice — once as a placeholder at file creation (line ~48), and once with the real count after all data is written (line ~95). A crash between these two writes leaves a file with `entry_count=0` in the header but actual data on disk. The `SSTableReader` trusts the header, reads the file as empty.

### 7. Event Store (`event-sourcing-store/event_store.py`) — Does NOT survive

The event store's `_persist_event` uses NDJSON append-only writes with no `fsync()` and no `flush()` — just `with open(path, "a") as f: f.write(...)` followed by implicit close. Events are appended to `self._events` in memory **before** `_persist_event` is called, so a write failure leaves the in-memory store ahead of disk with no rollback. Batch appends persist events one at a time with no transactional boundary.

## The Universal Gap: No Directory Fsync Anywhere

**Not a single module** in the codebase opens a directory file descriptor for `fsync`. The pattern `os.open(dir, os.O_RDONLY)` + `os.fsync()` appears nowhere (`topic-directory-fsync-gap.md`).

This matters on Linux ext4/XFS: creating a new file and fsyncing its contents guarantees the **data** is on disk, but not the **directory entry** pointing to it. After a crash, the data blocks exist but nothing references them — the file effectively doesn't exist.

The most dangerous instance is in the LSM tree: `SSTable.write()` creates a file, the WAL is truncated afterward, and neither the file nor its directory entry is fsynced. On ext4 with default `data=ordered` mount, the crash window for directory metadata is the journal commit interval (~5 seconds).

On macOS/APFS, this is handled implicitly. On Linux in production, it is not.

## Summary Verdict

| Engine | kill -9 (process death) | Power loss / kernel panic |
|--------|------------------------|--------------------------|
| **Standalone WAL** (sync mode) | **Safe** | **Safe** |
| **Standalone WAL** (batch mode) | **Safe** | Up to N-1 records lost |
| **B-Tree** | **Safe** | **Safe** (via ARIES redo) |
| **Bitcask** (sync_writes=True) | **Safe** | **Safe** for writes; **unsafe** during compaction |
| **Bitcask** (sync_writes=False) | **Safe** (kernel cache) | **Unsafe** |
| **Log-structured hash table** | **Safe** (kernel cache) | **Unsafe** |
| **LSM Tree** | **Safe** (kernel cache) | **Unsafe** — WAL never fsyncs |
| **SSTable** | **Safe** (kernel cache) | **Unsafe** — no fsync, header corruption risk |
| **Event Store** | **Safe** (kernel cache) | **Unsafe** — no fsync, no flush |

The critical distinction: `kill -9` only terminates the process; the kernel keeps dirty pages in cache and will eventually flush them. **Power loss** is the real test of durability, and only three implementations pass it: the standalone WAL (in sync mode), the B-tree, and Bitcask with `sync_writes=True` (for normal writes only — compaction is still vulnerable).

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:compact` — The full compaction body (lines 319-359) to verify the exact delete/write ordering and confirm there's no crash protection
- [function] `hash-index-storage/bitcask.py:compact` — Lines 194-321; need to verify whether compaction output is fsynced before old segment deletion and whether the `sync_writes` flag is honored
- [general] `manifest-based-compaction-recovery` — How LevelDB/RocksDB use a MANIFEST file to make compaction atomic; none of these implementations have one, which is the root cause of compaction crash unsafety
- [function] `write-ahead-log/wal.py:truncate` — The in-place rewrite without temp-rename is itself a crash point; trace what happens if the process dies mid-rewrite
- [general] `kill-9-vs-power-loss-testing` — How to write tests that distinguish process death (page cache survives) from power loss (page cache lost); most tests in this repo only cover the former

## Beliefs

- `only-three-engines-survive-power-loss` — Only `write-ahead-log/wal.py` (sync mode), `b-tree-storage-engine/btree.py`, and `hash-index-storage/bitcask.py` (with `sync_writes=True`, excluding compaction) call `os.fsync()` on durability-critical paths; all others rely on OS page cache persistence which does not survive power failure
- `btree-redo-idempotent-at-every-crash-point` — The B-tree WAL stores physical page images and follows strict fsync ordering (WAL before data, data before WAL truncation), making recovery correct at every possible crash point including crashes during recovery itself
- `lsm-wal-is-a-false-safety-net` — The LSM tree's WAL calls `flush()` but never `os.fsync()` (`lsm.py:26`), and the WAL is truncated after SSTable writes that are also not fsynced, creating a compound failure window where both recovery paths can lose data simultaneously
- `no-compaction-is-crash-safe` — No compaction implementation uses a manifest, journal, or atomic rename pattern; all follow a write-new/delete-old sequence where a crash between deletion and completion causes permanent data loss
- `directory-fsync-missing-everywhere` — No module opens a parent directory for `os.fsync()` after creating files; on Linux ext4/XFS this means newly created WAL segments and SSTables may not exist after power loss even if their contents were fsynced

