# Topic: How the combination of `_recover_seq_num`, `_open_latest`, and `replay` collectively guarantee no data loss after a crash

**Date:** 2026-05-29
**Time:** 08:33

# How `_recover_seq_num`, `_open_latest`, and `replay` Guarantee No Data Loss After a Crash

The WAL's crash-recovery guarantee rests on a simple invariant: **every mutation is durably written to disk before it's acknowledged, and on restart the system can reconstruct the exact sequence of operations that were acknowledged**. Three functions divide this responsibility.

## The Recovery Chain

### Step 1: `_recover_seq_num` — Restoring the Counter

When a new `WriteAheadLog` instance is created (simulating a restart after crash), the constructor at `wal.py:75` calls `_recover_seq_num()`.

This function (`wal.py:85–97`) scans **every** `.wal` file in sorted order, reading all valid records, and returns the highest `seq_num` it finds. This is critical because the sequence number is an in-memory counter (`self._seq_num`) that would otherwise reset to 0 on restart. Without this recovery:

- New writes would reuse sequence numbers already assigned to persisted records.
- `truncate(up_to_seq)` could accidentally delete new data that shares a sequence number with old data.
- `replay(after_seq=N)` would return wrong results because the sequence-number timeline would have gaps or collisions.

The scan stops cleanly on partial reads (`rec is None`) or CRC failures (`except ValueError: break`), meaning a half-written record from a crash is silently skipped — the counter only advances to the last **fully written, integrity-verified** record.

### Step 2: `_open_latest` — Resuming the Append Point

Called at `wal.py:78`, immediately after sequence recovery. This function (`wal.py:100–108`) finds the most recent WAL file and reopens it in **append mode** (`"ab"`) — but only if it hasn't exceeded `_max_file_size`. Otherwise it rotates to a fresh file.

This is the bridge between recovery and forward progress. By reopening the last file for appending rather than creating a new one, it ensures:

1. **No file gap**: Records written before the crash and records written after restart live in a contiguous set of files that `_wal_files()` returns in sorted order.
2. **No overwrite**: The `"ab"` mode guarantees new writes go to the end of the file, never overwriting existing records.
3. **Size-limit respect**: If the last file was already full, `_rotate()` creates a new sequentially-numbered file, maintaining the sorted ordering that `replay` depends on.

### Step 3: `replay` — Reconstructing State

`replay` (`wal.py:212`) reads back all committed records, giving the application layer everything it needs to rebuild in-memory state. Combined with the `after_seq` parameter, it supports **incremental replay** — if the application checkpointed at sequence N, it can call `replay(after_seq=N)` to get only the operations that happened after the checkpoint.

The key safety properties of replay:

- It iterates all WAL files in order (via `_wal_files()`), so records span file rotations seamlessly.
- CRC validation in `_read_record` (`wal.py:56–58`) rejects any record corrupted by a partial write during crash — you get exactly the set of operations that were fully persisted.
- It filters out COMMIT and CHECKPOINT marker records, returning only data-bearing operations (PUT/DELETE).

## How They Compose

The three functions form a pipeline that runs in the `__init__` constructor (`wal.py:68–78`):

```
__init__
  ├── _recover_seq_num()   →  "What's the last thing we durably wrote?"
  ├── _open_latest()       →  "Where do we append next?"
  └── (caller invokes)
      replay(after_seq=N)  →  "Give me everything since my last checkpoint"
```

The guarantee is: **if `append()` or `append_batch()` returned a sequence number to the caller, that record will survive a crash and appear in `replay()`**. This holds because:

1. `append` writes the record and calls `_do_sync()` (which calls `fsync` in sync mode) **before** returning the sequence number.
2. `_recover_seq_num` will find that record on restart because it scans all files and the record passed CRC validation at write time.
3. `_open_latest` positions the write cursor after that record, so it won't be overwritten.
4. `replay` will yield that record back to the application.

The `test_crash_recovery` test (`test_wal.py:30–41`) demonstrates this directly: it writes two records, then creates a **new** `WriteAheadLog` instance on the same directory (simulating a process restart), and verifies that both records are recovered and that the next sequence number continues from 3.

## The Corruption Boundary

One subtle point: the guarantee is "no loss of **acknowledged** data," not "no loss of any data." In `batch` or `none` sync modes, `_do_sync` may not fsync every write (`wal.py:128–135`). Records that were buffered in the OS page cache but not fsynced at crash time may be lost. The system is honest about this tradeoff — `append_batch` forces a sync (`force=True` at line 153), so batch atomicity is always guaranteed even when individual appends are not.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:truncate` — How old WAL segments are garbage-collected after a checkpoint, and the subtle re-opening logic at line 210
- [function] `write-ahead-log/wal.py:append_batch` — How the COMMIT record creates atomic multi-operation transactions and why `force=True` is essential
- [function] `log-structured-merge-tree/lsm.py:replay` — How the LSM tree consumes WAL records to rebuild its memtable, closing the loop on crash recovery
- [general] `fsync-durability-tradeoffs` — How the three sync modes (sync, batch, none) trade latency for durability, and what "acknowledged" really means in each mode
- [file] `write-ahead-log/wal.py` — The remaining 64 lines (201–264) containing `iterate`, `replay`, `close`, and the truncation rewrite logic

## Beliefs

- `wal-seq-num-monotonic` — Sequence numbers are strictly monotonically increasing within a WAL instance; `_recover_seq_num` ensures this invariant holds across restarts by scanning all files for the maximum
- `wal-crc-guards-replay` — CRC32 validation in `_read_record` causes both recovery and replay to stop at the first corrupted record, preventing half-written crash artifacts from entering application state
- `wal-append-mode-no-overwrite` — `_open_latest` opens files in `"ab"` (append-binary) mode, making it impossible for post-crash writes to overwrite pre-crash records
- `wal-batch-commit-always-fsynced` — `append_batch` calls `_do_sync(force=True)`, guaranteeing batch durability regardless of the configured sync mode, while individual `append` calls respect the sync mode setting
- `wal-file-ordering-invariant` — WAL files are zero-padded numerically named (`000001.wal`, `000002.wal`, ...) and `_wal_files` returns them sorted, which `replay` and `_recover_seq_num` depend on to process records in write order across rotations

