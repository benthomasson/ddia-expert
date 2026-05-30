# Topic: What happens when serialized data exceeds page_size? The max_keys heuristic uses a conservative estimate but doesn't account for variable-length keys

**Date:** 2026-05-29
**Time:** 10:43

The observations only include the first 200 lines of a 612-line file, so I'm missing the critical parts. Let me work with what's available and note the gaps.

## What Happens When Serialized Data Exceeds `page_size`

### The Serialization Format

The `_serialize_leaf` function (line ~173 in `btree.py`) builds a byte buffer by concatenating:
1. A 3-byte header (`HEADER_FMT = '>BH'` — 1 byte page type + 2 bytes key count)
2. A 4-byte next-sibling pointer
3. For each key-value pair: `2B key_len + key_bytes + 2B val_len + val_bytes`

Every field except the keys and values is fixed-width. The total serialized size is:

```
3 (header) + 4 (sibling ptr) + sum_over_entries(2 + len(key) + 2 + len(value))
```

This means two pages with the same `num_keys` can have wildly different serialized sizes depending on key/value lengths.

### The Truncation Safety Net (and Its Weakness)

In `PageManager.write_page` (lines ~80-87), there's a silent truncation guard:

```python
elif len(data) > self.page_size:
    data = data[:self.page_size]
```

If serialized data exceeds `page_size`, **the page is silently truncated**. No exception, no warning. The trailing key-value entries are simply chopped off. On the next read, `_deserialize_leaf` will try to unpack `num_keys` entries from the header, but the actual bytes for those entries won't be there — this will either crash with a `struct.unpack` error or return corrupted data depending on where the cut falls.

### The `max_keys_per_page` Heuristic

From the test file (line ~8), we see the BTree constructor accepts `max_keys_per_page=4`. The test `test_too_large_kv` (lines ~111-118) shows there *is* a guard for individual key-value pairs that are too large:

```python
tree = BTree(tmpdir, page_size=128, max_keys_per_page=4)
tree.put("x", b"y" * 200)  # raises ValueError
```

So the code validates individual entries against `page_size`. But `max_keys_per_page` is a static cap — it doesn't compute `4 * avg_entry_size` at insertion time. If you set `max_keys_per_page=4` with `page_size=128`, and insert four 25-byte keys with 25-byte values, the total payload is `4 * (2+25+2+25) = 216 bytes` plus 7 bytes of header = 223 bytes, far exceeding 128.

### The Gap

The per-entry size check catches the obvious case (single entry bigger than the page). But there's no check that validates whether the **aggregate** of `max_keys_per_page` entries actually fits. The split decision is triggered by key count, not by serialized byte count. This means:

1. With short keys (like the test's `"key_00001"`), `max_keys_per_page=4` works fine because each entry is ~20 bytes.
2. With variable-length keys approaching `page_size / max_keys_per_page`, the page overflows and gets silently truncated.
3. Production B-trees (InnoDB, SQLite, etc.) split on **byte fullness**, not key count, precisely to handle this.

**The observations are insufficient to fully answer this** — the remaining 412 lines of `btree.py` likely contain the `BTree` class with the split logic, the `put` method, and possibly additional size checks. The `_serialize_internal` function and the split threshold logic are not visible in the first 200 lines.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:BTree.put` — The insert path contains the split trigger; need to see whether it checks serialized size or just key count
- [function] `b-tree-storage-engine/btree.py:_serialize_internal` — Internal nodes have child pointers instead of values; their overflow characteristics differ
- [general] `byte-level-split-policy` — How production B-trees (SQLite's cell overflow pages, InnoDB's page split heuristics) avoid this problem by splitting on fill factor rather than key count
- [function] `b-tree-storage-engine/btree.py:PageManager.write_page` — The silent truncation at line ~84 is a data-loss bug; worth tracing what happens downstream when a truncated page is read back
- [file] `b-tree-storage-engine/test_btree.py` — The test suite uses short, fixed-length keys; adding a test with variable-length keys near the page boundary would expose the overflow

## Beliefs

- `write-page-silently-truncates` — `PageManager.write_page` silently truncates data exceeding `page_size` (line ~84) rather than raising an error, which can cause data loss
- `split-uses-key-count-not-bytes` — The B-tree splits nodes when key count reaches `max_keys_per_page`, not when serialized size approaches `page_size`, making it vulnerable to overflow with large keys
- `single-entry-size-validated` — Individual key-value pairs are validated against `page_size` before insertion (evidenced by `test_too_large_kv`), but aggregate size of all entries in a node is not checked
- `leaf-serialization-is-variable-length` — Leaf page serialization uses 2-byte length prefixes for both keys and values (`'>H'`), making actual page consumption dependent on key/value content, not just count

