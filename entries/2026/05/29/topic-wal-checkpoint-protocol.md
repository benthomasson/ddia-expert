# Topic: DDIA's discussion of how checkpoints bound recovery time and enable log truncation (Chapter 3, SSTables and LSM-Trees)

**Date:** 2026-05-29
**Time:** 11:17

No problem — the observations have enough to build a complete explanation. Here's the walkthrough.

---

# How Checkpoints Bound Recovery Time and Enable Log Truncation

## The Core Problem

Every write in an LSM tree first goes to a write-ahead log (WAL) before touching the in-memory memtable. This guarantees durability — if the process crashes, you replay the WAL to reconstruct state. But WAL replay has a cost proportional to log size: replay 1 million records and startup takes a while. DDIA's key insight is that **checkpoints create a known-good boundary** in the log, so recovery only needs to replay from the last checkpoint forward, and everything before it can be safely deleted.

## Two Implementations, Two Approaches

This codebase demonstrates both an **implicit checkpoint** (the LSM tree's flush) and an **explicit checkpoint record** (the standalone WAL module).

### Implicit Checkpoints: Memtable Flush as Checkpoint

In `log-structured-merge-tree/lsm.py`, the memtable flush **is** the checkpoint. When the memtable hits its size threshold (`lsm.py:244`), `_flush()` is called (`lsm.py:303`). The flush:

1. **Freezes** the current memtable and replaces it with a fresh one (`lsm.py:307-308`):
   ```python
   frozen = self._memtable
   self._memtable = SortedDict()
   ```
2. **Writes** the frozen memtable's contents to a new SSTable on disk (the durable checkpoint).
3. **Truncates** the WAL (`lsm.py:314`):
   ```python
   self._wal.truncate()
   ```

The truncation at `lsm.py:56-60` is total — it reopens the WAL file in write mode, erasing everything:
```python
def truncate(self):
    self._fd.close()
    self._fd = open(self._path, "wb")
    self._fd.close()
    self._fd = open(self._path, "ab")
```

This is safe because once the SSTable is fsynced to disk, the WAL entries that produced it are redundant. The SSTable **is** the checkpoint — it's a durable snapshot of everything the WAL contained.

**Recovery** (`test_lsm.py:92-99`) works by replaying whatever remains in the WAL after the last truncation. Since the WAL is truncated on every flush, recovery only replays entries written *since the last flush* — at most `memtable_threshold` entries (default 1000, per `lsm.py:202`). This is DDIA's point: **the memtable size threshold directly bounds recovery time**.

### Explicit Checkpoints: The WAL Module

The standalone WAL in `write-ahead-log/wal.py` takes the explicit approach — a dedicated `OP_CHECKPOINT` record type (`wal.py:7`) written into the log stream itself.

The `checkpoint()` method (`wal.py:169-177`) writes a checkpoint record with a force-sync:
```python
def checkpoint(self) -> int:
    self._seq_num += 1
    seq = self._seq_num
    self._fd.write(_encode_record(seq, OP_CHECKPOINT, b"", b""))
    self._do_sync(force=True)
```

This checkpoint sequence number then becomes the argument to `truncate(up_to_seq)` (`wal.py:179`), which removes all records with `seq_num <= up_to_seq`. Unlike the LSM tree's all-or-nothing truncation, this WAL supports **partial truncation** — it rewrites each WAL file keeping only records above the truncation point, and deletes files that become empty.

The protocol is:
1. Ensure application state is durable up to some point (e.g., SSTable flushed).
2. Call `checkpoint()` to get a sequence number.
3. Call `truncate(checkpoint_seq)` to discard everything the checkpoint covers.

## How This Bounds Recovery Time

Without checkpoints, recovery replays the *entire* WAL — every write since the database was created. With checkpoints:

| Approach | Recovery replays | Bounded by |
|----------|-----------------|------------|
| LSM implicit (`lsm.py`) | Entries since last flush | `memtable_threshold` (line 202) |
| Explicit WAL (`wal.py`) | Entries after `up_to_seq` | Checkpoint frequency |

The tradeoff DDIA highlights: **more frequent checkpoints mean faster recovery but more I/O during normal operation** (each flush writes an SSTable and fsyncs). The `memtable_threshold` parameter is the tuning knob — set it low for fast recovery, high for better write throughput.

## What's Missing

The observations don't show the LSM tree's `__init__` recovery path (the constructor logic that calls `self._wal.replay()` on startup). We can see from `test_crash_recovery` (`test_lsm.py:92-99`) that reopening an `LSMTree` on the same directory recovers unflushed writes, confirming replay happens, but the actual replay-and-reconstruct code in the constructor wasn't captured. The WAL module's `replay()` method for iterating records (`wal.py` beyond line 200) was also truncated.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:_flush` — Trace how frozen memtable becomes SSTable and how WAL truncation is ordered relative to the SSTable fsync (write ordering is critical for crash safety)
- [function] `write-ahead-log/wal.py:truncate` — The partial truncation logic that rewrites WAL files in place — study how it handles the current file vs. older rotated files
- [function] `write-ahead-log/wal.py:replay` — How replay interacts with `OP_COMMIT` records to ensure only committed batches are applied (partial batches without COMMIT should be discarded)
- [general] `checkpoint-frequency-tradeoffs` — DDIA discusses how checkpoint interval trades write amplification against recovery time; experiment with different `memtable_threshold` values and measure recovery duration
- [file] `write-ahead-log/test_wal.py` — The truncation test (line 60) and crash recovery test (line 29) demonstrate the checkpoint-then-truncate protocol in isolation

## Beliefs

- `lsm-wal-truncation-is-total` — The LSM tree's WAL truncation (`lsm.py:56`) erases the entire log file rather than truncating to a sequence number, because every flush checkpoints all in-flight data
- `wal-checkpoint-is-explicit-record-type` — The standalone WAL module uses a dedicated `OP_CHECKPOINT` opcode (value 4) written as a real record in the log stream, not out-of-band metadata
- `memtable-threshold-bounds-recovery` — The `memtable_threshold` parameter (`lsm.py:202`) directly caps the maximum number of WAL entries that must be replayed on crash recovery, because the WAL is truncated on every flush
- `wal-truncate-preserves-records-above-seq` — `WriteAheadLog.truncate(up_to_seq)` (`wal.py:179`) keeps records with `seq_num > up_to_seq` and deletes only those at or below, enabling partial log reclamation tied to checkpoint boundaries

