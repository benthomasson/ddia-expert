# Function: truncate_before in unbundled-database/unbundled_database.py

**Date:** 2026-05-29
**Time:** 06:39



## `WriteAheadLog.truncate_before`

### Purpose

This is a **log compaction** method. It discards WAL entries older than a given LSN (Log Sequence Number), reclaiming memory once those entries are no longer needed — typically after all consumers (derived systems) have processed past that point. This is the WAL equivalent of a checkpoint: once you know every downstream system has consumed through LSN *n*, entries before *n* are dead weight.

### Contract

- **Precondition**: `lsn` should be a valid LSN within the log's range (though no enforcement exists — see below).
- **Postcondition**: After the call, `self._entries` contains only entries with `lsn >= lsn`. The in-memory list is replaced, not mutated in place.
- **Invariant**: The relative ordering of retained entries is preserved. `_next_lsn` is unaffected — the LSN counter never rewinds.

### Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `lsn` | `int` | The cutoff. Entries with `lsn` **strictly less than** this value are discarded. Entries equal to or greater are kept. |

Edge cases:
- `lsn = 0` or `lsn <= earliest_lsn`: No entries are removed; returns 0.
- `lsn > latest_lsn`: **All** entries are removed. The WAL becomes empty, but `_next_lsn` still points past the last assigned LSN, so future appends continue with correct numbering.

### Return Value

Returns an `int` — the count of entries removed. The caller can use this for logging/metrics but has no obligation to act on it.

### Algorithm

1. Snapshot the current entry count (`before`).
2. Rebuild `_entries` via list comprehension, keeping only entries where `e.lsn >= lsn`.
3. Return the difference: `before - len(self._entries)`.

This is O(n) in the number of entries — it scans the entire list. Since entries are ordered by LSN (they're appended sequentially), a bisect + slice would be O(log n + k) where k is the number retained, but the simplicity here is appropriate for a reference implementation.

### Side Effects

- **Mutates `self._entries`**: The old list is replaced (not mutated), so any external reference to the old list object would still see the untruncated data. In practice, nothing holds such a reference.
- **Does NOT touch the persist file**: If the WAL was constructed with a `persist_path`, the on-disk file still contains the truncated entries. On next load, those entries will reappear. This is a memory-only truncation — a significant assumption that callers should be aware of.

### Error Handling

None. No exceptions are raised. Passing a negative LSN or a non-integer would either silently do nothing (negative) or raise a `TypeError` from the comparison operator — but neither case is guarded.

### Usage Patterns

Typical usage is checkpoint-driven truncation:

```python
# Find the lowest position across all consumers
min_position = min(sys.position for sys in derived_systems)
# Safe to discard everything consumers have already processed
removed = wal.truncate_before(min_position)
```

The caller is responsible for ensuring no consumer still needs entries below the cutoff. Truncating too aggressively means a consumer that falls behind cannot catch up — it would need a full rebuild from the storage engine instead of incremental replay.

### Dependencies

- Relies solely on `self._entries` (a `list[WALEntry]`) and the `lsn` attribute on each entry.
- No external module dependencies.
- Implicitly depends on entries being appended in LSN order, which `append()` guarantees.

---

## Topics to Explore

- [function] `unbundled-database/unbundled_database.py:CDCStream.snapshot_and_stream` — The recovery path when a consumer joins late or after truncation has discarded its needed history
- [function] `unbundled-database/unbundled_database.py:StorageEngine.rebuild` — How state is reconstructed from the WAL, and what happens if the WAL was truncated before rebuild is called
- [function] `unbundled-database/unbundled_database.py:WriteAheadLog.read_from` — The read-side counterpart to truncate_before; together they define the WAL's windowed access pattern
- [general] `log-compaction-vs-persistence` — The gap between in-memory truncation and on-disk state: how a real system would handle WAL segment files and checkpointing
- [file] `unbundled-database/test_unbundled_database.py` — Test coverage for truncation edge cases (empty WAL, truncate past end, interaction with persistence)

## Beliefs

- `wal-truncate-memory-only` — `truncate_before` removes entries from memory but does not modify the on-disk WAL file, so reloading from `persist_path` restores truncated entries
- `wal-truncate-preserves-lsn-counter` — Truncation never resets `_next_lsn`, so appending after truncation continues with monotonically increasing LSNs
- `wal-truncate-keeps-gte` — The cutoff is inclusive of the given LSN: entries with `lsn >= cutoff` are retained, entries with `lsn < cutoff` are discarded
- `wal-entries-ordered-by-lsn` — WAL entries in `_entries` are always in ascending LSN order, maintained by the sequential nature of `append()`

