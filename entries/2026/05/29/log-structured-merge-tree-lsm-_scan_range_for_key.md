# Function: _scan_range_for_key in log-structured-merge-tree/lsm.py

**Date:** 2026-05-29
**Time:** 13:18

## `SSTable._scan_range_for_key`

### Purpose

This is the low-level linear scan that resolves a key lookup within a bounded region of an SSTable file. The sparse index narrows a lookup to a small byte range (typically covering ~16 entries), and `_scan_range_for_key` does the final sequential walk through that range to find (or rule out) the target key.

It exists because SSTables use a **sparse index** — not every key has an index entry. The sparse index gets you close; this method finishes the job.

### Contract

**Preconditions:**
- `self.path` is a valid SSTable file on disk
- `start` and `end` are byte offsets that bracket a contiguous run of length-prefixed entries written by `SSTable.write`
- Entries within `[start, end)` are sorted by key in ascending lexicographic order (guaranteed by the SSTable write path, which receives entries from a `SortedDict`)

**Postconditions:**
- Returns `(True, value_bytes)` if the key exists in the scanned range — including tombstones (`TOMBSTONE = b""`)
- Returns `(False, None)` if the key is not in the range

**Invariant:** The file is not modified. The method is read-only and stateless.

### Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `key` | `str` | The key to search for |
| `start` | `int` | Byte offset where scanning begins (inclusive) |
| `end` | `int` | Byte offset where scanning stops (exclusive) — the cursor must stay below this |

`start` is typically from the sparse index entry. `end` is either the next sparse index entry's offset or the footer start offset (for the last segment).

### Return Value

`Tuple[bool, Optional[bytes]]`

- `(True, v)` — key found; `v` is the raw value bytes. The caller (`SSTable.get`) must check whether `v == TOMBSTONE` to distinguish a live value from a deletion marker.
- `(False, None)` — key not present in this range.

The caller does **not** need to handle exceptions from this method under normal operation — malformed data causes `_read_entry` to return `None`, which triggers a clean exit.

### Algorithm

1. **Open the file** at `self.path` in binary read mode.
2. **Seek** to `start`.
3. **Loop** while the file cursor is before `end`:
   - Read one entry via `_read_entry(f)`, which decodes a `(key_len, key, value_len, value)` record.
   - If `_read_entry` returns `None` (truncated/corrupt data), break.
   - If the entry's key matches `key`, return `(True, value)` immediately.
   - If the entry's key is **greater** than `key`, break early — since entries are sorted, the key cannot appear later. This is the critical optimization: it avoids scanning the remainder of the range.
4. **Fall through** to `return (False, None)`.

The early-exit on `k > key` turns worst-case from O(interval) to average O(interval/2) for misses, and makes cache-unfriendly sequential I/O shorter.

### Side Effects

- **I/O:** Opens and reads from `self.path`. The file handle is scoped to the `with` block and closed on exit.
- **No mutations:** No writes, no state changes to `self`.

### Error Handling

There is no explicit exception handling. The method relies on:
- `_read_entry` returning `None` for short reads (graceful degradation on truncated files)
- The `with` block ensuring the file handle is closed even if an unexpected exception propagates

If the file is missing or unreadable, `open()` raises `FileNotFoundError` or `OSError`, which propagates to the caller unhandled.

**Assumption not enforced by the type system:** `start` and `end` must be valid byte offsets pointing to entry boundaries. Arbitrary offsets will cause `_read_entry` to parse garbage, likely returning `None` and silently reporting the key as not found.

### Usage Patterns

Called exclusively by `SSTable.get`, which computes `start` and `end` from the sparse index:

```python
def get(self, key):
    # ... bisect sparse index to find idx ...
    start_off = self._sparse_index[idx][1]
    end_off = self._sparse_index[idx + 1][1] if idx + 1 < len(...) else self._footer_start()
    return self._scan_range_for_key(key, start_off, end_off)
```

The caller never needs to validate the return — the tuple is always well-formed.

### Dependencies

- `self._read_entry(f)` — static method that decodes one `(key, value)` record from a file handle using `struct.unpack` with big-endian 4-byte length prefixes.
- Standard library: `struct` (for the entry format), `os`/file I/O.
- No external packages.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:SSTable.get` — The caller that translates a sparse index bisect into the byte range this method scans
- [function] `log-structured-merge-tree/lsm.py:SSTable.write` — How entries and the sparse index footer are serialized, which defines the on-disk format this method reads
- [function] `log-structured-merge-tree/lsm.py:SSTable._read_entry` — The record decoder; understanding its failure modes explains when `_scan_range_for_key` silently returns not-found
- [general] `sparse-vs-dense-sstable-indexing` — Trade-off between index size (memory) and scan cost (I/O) controlled by `sparse_index_interval`
- [function] `log-structured-merge-tree/lsm.py:LSMTree.get` — The full read path: memtable → immutable memtables → SSTables (newest-first), showing how this scan fits into the layered lookup

---

## Beliefs

- `scan-range-sorted-order-assumption` — `_scan_range_for_key` assumes entries between `start` and `end` are sorted by key ascending; it breaks early on `k > key`, so unsorted data causes silent false negatives
- `scan-range-tombstone-transparent` — `_scan_range_for_key` returns tombstone values (`b""`) as found; it does not interpret them — tombstone filtering is the caller's responsibility
- `scan-range-no-state-mutation` — `_scan_range_for_key` is a pure read operation with no side effects on `SSTable` instance state
- `scan-range-corrupt-data-silent` — If `_read_entry` encounters truncated data mid-range, `_scan_range_for_key` silently returns `(False, None)` rather than raising an error

