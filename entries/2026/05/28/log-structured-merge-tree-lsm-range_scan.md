# Function: range_scan in log-structured-merge-tree/lsm.py

**Date:** 2026-05-28
**Time:** 18:26

# `LSMTree.range_scan`

## Purpose

`range_scan` performs a multi-source merge query across all layers of the LSM tree — active memtable, immutable memtables, and on-disk SSTables — to return a consistent, deduplicated snapshot of key-value pairs within a key range. It exists because an LSM tree scatters data across multiple structures at different ages, and a range query must reconcile all of them, respecting write ordering so that newer writes shadow older ones and tombstones suppress deleted keys.

## Contract

**Preconditions:**
- `start_key < end_key` (not enforced — if reversed, the result is an empty list since no key satisfies the range predicate).
- Keys stored in the tree are UTF-8 strings; values are UTF-8 strings (encoded as bytes internally).

**Postconditions:**
- Returns pairs sorted in lexicographic key order.
- Each key appears at most once, reflecting the most recent write.
- Deleted keys (tombstoned) are excluded from the result.
- The range is half-open: `[start_key, end_key)`.

**Invariants relied upon:**
- `self._sstables` is ordered oldest-first (by sequence number).
- `self._immutable_memtables` is ordered oldest-first.
- The active memtable is always the newest source.

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `start_key` | `str` | Inclusive lower bound of the scan range. Must be a valid string comparable to stored keys via Python's default string ordering. |
| `end_key` | `str` | Exclusive upper bound. Keys equal to `end_key` are **not** returned. |

**Edge cases:** If `start_key == end_key`, the range is empty. If no keys fall within the range across any layer, the result is `[]`.

## Return Value

`List[Tuple[str, str]]` — a list of `(key, value)` pairs sorted by key. Values are decoded from bytes to UTF-8 strings. The caller receives a complete snapshot; there is no cursor or iterator to manage.

If every matching key has been deleted (tombstoned), the result is an empty list — the caller cannot distinguish "no keys in range" from "all keys deleted."

## Algorithm

The method uses a **priority-based last-writer-wins merge**:

1. **Initialize** a `merged` dict mapping each key to `(priority, value_bytes)`. Priority is an integer counter incremented per source, oldest sources first.

2. **Scan SSTables** (oldest → newest). For each SSTable, call `sst.scan(start_key, end_key)` which yields `(key, value_bytes)` for keys in `[start_key, end_key)`. Each result overwrites any prior entry in `merged` for that key, because the priority is higher. This is the critical insight: since sources are visited oldest-first and `merged[k] = (priority, v)` is a plain overwrite, the last write to any key naturally wins.

3. **Scan immutable memtables** (oldest → newest). Uses `SortedDict.irange()` with bounds `(True, False)` meaning inclusive-start, exclusive-end. Same overwrite logic.

4. **Scan the active memtable** (newest of all). Same `irange` pattern, highest priority.

5. **Build the result list.** Iterate `merged` in sorted key order. Skip any entry whose value is `TOMBSTONE` (the empty bytes sentinel `b""`). Decode surviving values to UTF-8 strings.

The priority value is tracked but only the *value* paired with it matters — the priority field itself is never compared or used for tie-breaking. Its only role is documentation: the overwrite order guarantees correctness, and the priority integer is vestigial (a leftover from a design that might have used `max()` instead of overwrite).

## Side Effects

**None.** This is a pure read operation. It does not modify the memtable, WAL, SSTables, or any tree state. SSTable scans open and close file handles internally (`sst.scan` uses `open` within a `with` block).

## Error Handling

No explicit error handling. Possible exceptions:
- `FileNotFoundError` / `IOError` — if an SSTable file is missing or corrupted during `sst.scan()`. Not caught here; propagates to the caller.
- `UnicodeDecodeError` — if a stored value is not valid UTF-8. The `.decode("utf-8")` in the result-building loop would raise.
- No protection against concurrent modification (e.g., a flush or compaction deleting an SSTable mid-scan).

## Usage Patterns

```python
tree = LSMTree("/tmp/data")
tree.put("user:001", "alice")
tree.put("user:002", "bob")
tree.put("user:003", "charlie")
results = tree.range_scan("user:001", "user:003")
# [("user:001", "alice"), ("user:002", "bob")]
```

Callers use this for prefix scans (e.g., all keys starting with `"user:"`) by computing an appropriate `end_key`. The caller must understand that the result is a point-in-time snapshot with no isolation guarantees — concurrent writes may or may not be visible.

## Dependencies

- **`SSTable.scan`** — yields key-value pairs from a single on-disk table within a key range, using the sparse index for seek optimization.
- **`SortedDict.irange`** (from `sortedcontainers`) — efficient range iteration over the in-memory sorted dict. The `(True, False)` tuple controls inclusive/exclusive bounds.
- **`TOMBSTONE`** (`b""`) — module-level sentinel marking deleted keys.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:SSTable.scan` — How the sparse index accelerates range scans within a single SSTable, and what happens when `start_key` falls between index entries
- [function] `log-structured-merge-tree/lsm.py:LSMTree.compact` — How compaction merges SSTables and removes tombstones permanently, reducing the number of sources `range_scan` must consult
- [function] `log-structured-merge-tree/lsm.py:LSMTree.get` — Contrast the point-lookup strategy (newest-first short-circuit) with range_scan's oldest-first overwrite approach — both achieve last-writer-wins but with opposite iteration orders
- [general] `lsm-concurrency-safety` — This implementation has no locking; explore what breaks if `_flush()` or `compact()` runs concurrently with `range_scan`
- [general] `merge-iterator-vs-dict-merge` — Production LSM trees (LevelDB, RocksDB) use a heap-based merge iterator instead of materializing all results into a dict; explore the memory and latency tradeoffs

## Beliefs

- `range-scan-last-writer-wins` — `range_scan` iterates sources oldest-to-newest so that dict overwrites naturally give each key the value from the newest source containing it
- `range-scan-half-open-interval` — The scan range is `[start_key, end_key)`: inclusive on start, exclusive on end, enforced by both `SSTable.scan` and `SortedDict.irange`
- `range-scan-pure-read` — `range_scan` has no side effects: it does not mutate tree state, trigger flushes, or modify the WAL
- `range-scan-priority-field-unused` — The priority integer stored in `merged` is never read or compared; correctness relies entirely on the overwrite order, making the priority field dead weight
- `range-scan-no-concurrency-protection` — `range_scan` can observe inconsistent state if a flush or compaction runs concurrently, since there is no snapshot isolation or locking

