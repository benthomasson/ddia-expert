# Topic: Consider what would break if `_flush` were made async or threaded without the immutable memtable fix

**Date:** 2026-05-29
**Time:** 14:22

The observations provided give me what I need — the full `_flush` path is visible through the grep results. Let me piece together the critical analysis.

---

# What Breaks If `_flush` Goes Async Without the Immutable Memtable

## The Current Design

Look at `_flush` in `log-structured-merge-tree/lsm.py:303-311`. The method does three things in sequence:

1. **Line 305**: Guard — bail if memtable is empty
2. **Line 307**: `frozen = self._memtable` — capture a reference to the current memtable
3. **Line 308**: `self._memtable = SortedDict()` — swap in a fresh, empty memtable
4. **Line 311**: `entries = list(frozen.items())` — serialize the frozen data to an SSTable

This is a **swap-and-write** pattern. The active memtable is replaced instantly (line 308), and the old one is written to disk from the `frozen` reference. Because `_flush` runs synchronously, the frozen memtable exists only as a local variable — it's never visible to concurrent readers.

## The Danger: What's Missing

Notice that `_immutable_memtables` (line 210) exists as a list, and the read path already checks it (lines 253-254):

```python
# Check immutable memtables (newest first)
for mt in reversed(self._immutable_memtables):
```

But **`_flush` never adds `frozen` to `_immutable_memtables`**. It goes straight from `frozen = self._memtable` (line 307) to writing the SSTable. This works only because `_flush` is synchronous — the frozen memtable is fully written to an SSTable and the SSTable is added to `self._sstables` before any read can interleave.

## Three Failure Modes If You Make `_flush` Async

### 1. Lost reads during flush (data invisibility window)

If `_flush` yields or runs on a thread, a `get()` call arriving mid-flush would:
- Check `self._memtable` (line 250) — the **new, empty** one (swapped at line 308)
- Check `self._immutable_memtables` (line 254) — **empty**, because `frozen` was never added
- Check SSTables — the flush hasn't finished writing yet

Result: the key appears to not exist. Data that was just written is temporarily invisible. This is a **read anomaly** — the system violates its own consistency guarantees silently.

### 2. Lost reads during range scans

The `scan` method (lines 285-293) has the same structure: it merges immutable memtables, then the active memtable, then SSTables. A scan running concurrently with an async flush would skip the entire frozen memtable's worth of data. Since scans are used for compaction input, this could cause **data loss**, not just read anomalies.

### 3. Mutation of `frozen` during write

If `_flush` runs on a thread while the main thread continues accepting writes, `self._memtable` at line 307 captures a **reference**, not a copy. If another `_flush` is triggered before the first finishes (e.g., the new memtable fills up quickly), you'd get:
- Line 307 of the second flush: `frozen = self._memtable` — captures the second memtable
- But the first flush is still iterating `frozen.items()` (line 311) on the **first** memtable

This particular scenario is safe because the two `frozen` variables are in different stack frames. But if the write path mutated the old memtable (e.g., an upsert that checks "does this key already exist in a pending flush?"), you'd get concurrent reads and writes on a `SortedDict` with no lock.

## The Fix

The "immutable memtable fix" would be:

```python
def _flush(self):
    if not self._memtable:
        return
    frozen = self._memtable
    self._immutable_memtables.append(frozen)  # <-- THE FIX
    self._memtable = SortedDict()
    # ...write SSTable async...
    self._immutable_memtables.remove(frozen)   # <-- after SSTable is live
```

By parking `frozen` in `_immutable_memtables` before yielding control, the read path (which already iterates `self._immutable_memtables` at line 254) would find the data during the flush window. The infrastructure for this is already wired into `get()` and `scan()` — it just isn't used by `_flush`.

This is a textbook example of DDIA's principle that LSM trees need immutable memtables as a bridge between the write path and the read path during flushes. The current code gets away without it because synchronous execution provides an implicit barrier.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:scan` — The merge logic across memtables, immutable memtables, and SSTables reveals the full read-path priority order
- [function] `log-structured-merge-tree/lsm.py:_compact` — Compaction reads SSTables via `scan_all`, and would also miss data during an async flush if immutable memtables aren't consulted
- [file] `log-structured-merge-tree/test_lsm.py` — Tests use `memtable_threshold=2` or `3` to force flushes; check whether any test would catch the async race
- [general] `wal-vs-immutable-memtable-recovery` — The WAL provides crash recovery but not concurrency safety — understand the distinct roles of WAL replay (line 30) vs immutable memtable reads
- [file] `hash-index-storage/bitcask.py` — Lines 196-204 show a similar immutable/active split for compaction in Bitcask, but applied to file segments rather than memtables

## Beliefs

- `lsm-flush-not-using-immutable-list` — `_flush` assigns `frozen = self._memtable` but never appends it to `self._immutable_memtables`, relying on synchronous execution to avoid a read visibility gap
- `lsm-read-path-checks-immutable-memtables` — `get()` at line 254 and `scan()` at line 286 already iterate `self._immutable_memtables`, so the read-path infrastructure for concurrent flushes exists but is unused
- `lsm-memtable-swap-is-reference-not-copy` — Line 307 `frozen = self._memtable` captures a reference to the SortedDict, not a deep copy, meaning concurrent mutation of the dict would be unsafe without synchronization
- `lsm-flush-synchronous-invariant` — The correctness of the LSM tree depends on `_flush` completing (memtable swap through SSTable registration) without any interleaved read or write operations

