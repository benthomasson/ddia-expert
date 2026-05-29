# Topic: The page size is fixed but serialization has no overflow guard; trace how `BTree.put` and `max_keys` cooperate to prevent corruption

**Date:** 2026-05-29
**Time:** 07:47

The observations only include the first 200 of 612 lines of `btree.py` — the `BTree` class itself (including `put`, split logic, and `max_keys` enforcement) is in the unread portion. I can trace the mechanism partially and will note what's missing.

---

## How `max_keys` and `put` Prevent Page Overflow

### The problem

Serialization functions like `_serialize_leaf` (`btree.py:173`) pack keys and values into a byte buffer with **no size check** against the page size. They just keep appending `len(key) + key + len(value) + value` for every entry:

```python
def _serialize_leaf(keys, values, next_sibling=NO_SIBLING):
    buf = struct.pack(HEADER_FMT, LEAF, len(keys))
    buf += struct.pack('>I', next_sibling)
    for k, v in zip(keys, values):
        kb = k.encode('utf-8') if isinstance(k, str) else k
        buf += struct.pack('>H', len(kb)) + kb
        buf += struct.pack('>H', len(v)) + v
    return buf
```

If the buffer exceeds `page_size`, `PageManager.write_page` (`btree.py:79-84`) **silently truncates**:

```python
elif len(data) > self.page_size:
    data = data[:self.page_size]
```

That truncation would corrupt the node — the header would claim `num_keys` entries exist, but deserialization would hit truncated or missing bytes. So something upstream must ensure the buffer never grows too large.

### The defense: two layers

**Layer 1 — `max_keys_per_page` caps node size.** The tests reveal this parameter (`test_btree.py:9`):

```python
tree = BTree(tmpdir, max_keys_per_page=4)
```

When a node reaches `max_keys_per_page` entries, `put` splits the node *before* adding the new key, keeping each node's entry count at or below the limit. With a bounded key count and bounded individual key/value sizes, the serialized output stays within `page_size`.

**Layer 2 — Individual KV size validation.** The test `test_too_large_kv` (`test_btree.py:107-114`) proves there's an upfront size check:

```python
tree = BTree(tmpdir, page_size=128, max_keys_per_page=4)
try:
    tree.put("x", b"y" * 200)
    assert False, "should have raised ValueError"
except ValueError:
    pass
```

A single key-value pair that can't fit in a page (even alone) raises `ValueError` before any write occurs. This blocks the degenerate case where one entry alone exceeds page capacity.

### The invariant

The two layers cooperate to maintain this invariant:

> **For any node with ≤ `max_keys_per_page` entries where each entry individually fits in a page, the serialized node fits in `page_size` bytes.**

The `max_keys_per_page` value must be set conservatively enough relative to `page_size` and maximum expected key/value sizes. The default `page_size=4096` with reasonable key/value sizes gives ample room. The test suite uses `page_size=128` with `max_keys_per_page=4` to stress the boundary.

### What's missing from these observations

The actual `BTree.put` method, the split logic, and the `max_keys` check are in lines 200-612 of `btree.py` — the unread portion. To fully trace the mechanism, you'd want to see:

1. **Where `put` checks `len(keys) >= max_keys_per_page`** and triggers a split
2. **The split functions** (`_split_leaf`, `_split_internal` or similar) — how they divide keys and allocate new pages
3. **The `ValueError` guard** — the exact check that rejects oversized individual KV pairs
4. **How `max_keys_per_page` relates to `page_size`** — whether there's a formula or it's a caller-provided constant

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:BTree.put` — The full insertion path: where it checks key count, triggers splits, and validates entry size before writing
- [function] `b-tree-storage-engine/btree.py:_serialize_internal` — Internal nodes serialize child pointers alongside keys; understanding the space budget differs from leaves
- [general] `split-propagation` — How a leaf split propagates upward when the parent is also full, and how root splits increase tree height
- [function] `b-tree-storage-engine/btree.py:PageManager.write_page` — The silent truncation at line 83 is a latent corruption vector; trace whether any code path can actually reach it post-validation
- [file] `b-tree-storage-engine/test_btree.py` — The test suite exercises `page_size=128` with `max_keys_per_page=4`, which is the tightest fit — study how those numbers were chosen

---

## Beliefs

- `serialization-has-no-overflow-check` — `_serialize_leaf` packs all entries into a buffer without comparing its length to `page_size`; overflow prevention is entirely the caller's responsibility
- `write-page-silently-truncates` — `PageManager.write_page` truncates data exceeding `page_size` to exactly `page_size` bytes without raising an error, which would corrupt the node header's `num_keys` count
- `max-keys-prevents-overflow` — The `max_keys_per_page` parameter bounds entries per node so that serialized output fits within `page_size`, with splits triggered before the limit is exceeded
- `single-kv-size-validated` — `BTree.put` raises `ValueError` for any single key-value pair that cannot fit in a page, blocking the case where even one entry would overflow

