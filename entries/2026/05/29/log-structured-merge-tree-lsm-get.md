# Function: get in log-structured-merge-tree/lsm.py

**Date:** 2026-05-29
**Time:** 14:21

## `SSTable.get` — Point Lookup via Sparse Index

### Purpose

This method performs a point lookup for a single key within a sorted string table (SSTable). SSTables store key-value pairs in sorted order on disk, and this method uses a sparse index — a sampled subset of keys with their file offsets — to narrow the search region before doing a linear scan. It exists because reading the entire SSTable sequentially for every lookup would be prohibitively slow; the sparse index turns an O(n) scan into an O(n/k) scan where k is the index interval (default 16).

### Contract

**Preconditions:**
- `key` is a valid UTF-8 string.
- If the SSTable was loaded from disk (not freshly written), `load_index()` must have been called first to populate `_sparse_index`. The method does not call `load_index()` itself — it silently returns `(False, None)` on an empty index.

**Postconditions:**
- Returns `(True, value_bytes)` if the key exists in this SSTable, where `value_bytes` may be `TOMBSTONE` (empty bytes `b""`).
- Returns `(False, None)` if the key is not present.

**Invariant:** The sparse index entries are in sorted key order, matching the on-disk sort order.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `str` | The key to search for. Must match the encoding used at write time (UTF-8). |

### Return Value

`Tuple[bool, Optional[bytes]]` — A two-element tuple:
- First element: whether the key was found.
- Second element: the raw value bytes if found, `None` otherwise. Critically, a found key may have a `TOMBSTONE` value (`b""`), meaning it was deleted. The caller must check for this — `get` does **not** filter tombstones.

### Algorithm

1. **Empty index guard** — If `_sparse_index` is empty (no data or `load_index()` never called), return miss immediately.

2. **Extract index keys** — Build a list of just the keys from the sparse index (dropping offsets). This is O(m) where m is the number of index entries (~total_entries / 16).

3. **Binary search** — Use `bisect.bisect_right(keys, key) - 1` to find the rightmost index entry whose key is ≤ the target. `bisect_right` returns the insertion point *after* any equal keys, so subtracting 1 gives the last entry that could be a prefix of the range containing our key.

4. **Clamp to zero** — If `idx < 0` (the target key is smaller than every indexed key), clamp to 0. This is correct: the target might still exist between the start of file and the first indexed key.

5. **Compute scan range** — The scan region runs from `_sparse_index[idx][1]` (the file offset of the index entry) to either the next index entry's offset or the footer start (if this is the last index entry). This bounds the linear scan to at most `sparse_index_interval` entries.

6. **Linear scan** — Delegate to `_scan_range_for_key`, which reads entries sequentially within `[start_off, end_off)`, returning on exact match or early-exiting when it passes the target key (since entries are sorted).

### Side Effects

- **Disk I/O**: `_scan_range_for_key` opens and reads from the SSTable file. If this is the last index segment, `_footer_start()` also opens the file to read the footer pointer — that's two file opens for a single lookup in the worst case.
- No mutations to in-memory state.

### Error Handling

No explicit error handling. The method will raise:
- `FileNotFoundError` if the SSTable file was deleted between index load and lookup.
- `struct.error` if the file is corrupted (truncated records).
- `UnicodeDecodeError` if a key on disk isn't valid UTF-8.

None of these are caught — they propagate to the caller.

### Usage Patterns

Called by `LSMTree.get`, which searches SSTables from newest to oldest:

```python
for sst in reversed(self._sstables):
    found, v = sst.get(key)
    if found:
        return None if v == TOMBSTONE else v.decode("utf-8")
```

The caller is responsible for:
1. Interpreting `TOMBSTONE` as a deletion (not returning `b""` as a valid value).
2. Searching SSTables in reverse sequence order so newer writes shadow older ones.
3. Ensuring `load_index()` was called after opening an existing SSTable from disk.

### Dependencies

- `bisect.bisect_right` — standard library binary search.
- `self._sparse_index` — populated by either `SSTable.write` (at creation) or `load_index()` (from disk).
- `self._scan_range_for_key` — does the actual sequential read within the narrowed byte range.
- `self._footer_start` — reads the footer pointer from the last 12 bytes of the file to know where data entries end.

### Notable Assumptions

1. **Keys in `_sparse_index` are sorted.** The binary search is only correct if this holds. It's guaranteed by construction (`write` iterates a pre-sorted list, `load_index` reads them in file order), but there's no runtime assertion.

2. **The scan range `[start_off, end_off)` contains all keys between two consecutive index keys.** This relies on the data being written in sorted order with no gaps — valid for a properly constructed SSTable, but not validated at read time.

3. **Extracting keys into a list every call** (`keys = [k for k, _ in self._sparse_index]`) is O(m) allocation on every lookup. For a hot path, this could be cached. The current implementation trades memory for simplicity.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:_scan_range_for_key` — The linear scan that does the actual record-level search within the narrowed byte range
- [function] `log-structured-merge-tree/lsm.py:LSMTree.get` — How the LSM tree orchestrates lookups across memtable, immutable memtables, and SSTables with newest-wins semantics
- [function] `log-structured-merge-tree/lsm.py:SSTable.write` — How the sparse index is constructed during SSTable creation and why the interval parameter controls the space/time tradeoff
- [general] `bloom-filter-before-sstable-lookup` — How production LSM trees (LevelDB, RocksDB) use bloom filters to skip SSTables entirely, avoiding the disk I/O this method always performs
- [function] `log-structured-merge-tree/lsm.py:compact` — How compaction merges SSTables and eliminates tombstones, reducing the number of SSTables this method might need to search

---

## Beliefs

- `sstable-get-does-not-filter-tombstones` — `SSTable.get` returns `(True, TOMBSTONE)` for deleted keys; tombstone interpretation is the caller's responsibility
- `sparse-index-bisect-narrows-to-interval-range` — The binary search on the sparse index reduces the linear scan to at most `sparse_index_interval` entries (default 16)
- `sstable-get-requires-prior-load-index` — For SSTables loaded from disk, `load_index()` must be called before `get()` or every lookup silently returns `(False, None)`
- `sstable-get-rebuilds-key-list-every-call` — The sparse index keys are extracted into a fresh list on every `get()` call rather than being cached, making each lookup O(m) in index size before the binary search
- `sparse-index-sorted-order-invariant` — `SSTable.get` assumes `_sparse_index` keys are in ascending sorted order; a violated invariant would silently produce wrong results rather than raising an error

