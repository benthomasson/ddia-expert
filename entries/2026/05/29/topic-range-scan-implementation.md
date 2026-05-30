# Topic: Follow the range scan code path to see how it walks the leaf chain using `next_sibling`, which is the actual use of the sibling links

**Date:** 2026-05-29
**Time:** 10:52

The observations only show the first 200 lines of `btree.py` (which is 612 lines total), so I'm missing the `range_scan` implementation at line 486 and the end of `_deserialize_leaf`. Let me work with what we have and note what's missing.

---

# Range Scan: Walking the Leaf Chain via `next_sibling`

## The Big Picture

A B-tree range scan has two phases: **find the starting leaf** (top-down tree traversal) and **walk the leaf chain** (horizontal traversal via sibling pointers). The sibling links exist specifically for this second phase — they let you scan consecutive key ranges without climbing back up to internal nodes.

## How Sibling Links Are Stored

Every leaf page carries a `next_sibling` pointer — a 4-byte page number written immediately after the page header:

**`btree.py:182-190`** — `_serialize_leaf` writes the leaf format:
```python
def _serialize_leaf(keys, values, next_sibling=NO_SIBLING):
    buf = struct.pack(HEADER_FMT, LEAF, len(keys))
    buf += struct.pack('>I', next_sibling)       # <-- sibling pointer, right after header
    for k, v in zip(keys, values):
        ...
```

**`btree.py:193-200`** — `_deserialize_leaf` reads it back:
```python
def _deserialize_leaf(data):
    """Returns (keys, values, next_sibling)."""
    _, num_keys = struct.unpack(HEADER_FMT, data[:HEADER_SIZE])
    offset = HEADER_SIZE
    next_sib = struct.unpack('>I', data[offset:offset+4])[0]
    offset += 4
    ...
```

The sentinel value `NO_SIBLING = 0xFFFFFFFF` (line 13) marks the rightmost leaf — the end of the chain.

## The Range Scan Code Path

The `range_scan` method is defined at `btree.py:486`. While the full implementation isn't in the observations, we can reconstruct the algorithm from the data structures and tests:

**Phase 1 — Find the start leaf.** The tree uses `_search` (its top-down traversal) to navigate from the root through internal nodes, using `bisect_left`/`bisect_right` (imported at line 6) to pick child pointers at each level. This descends to the leaf containing `start_key`.

**Phase 2 — Walk the leaf chain.** Once at the starting leaf:
1. Deserialize the leaf via `_deserialize_leaf`, getting `(keys, values, next_sibling)`
2. Collect all key-value pairs where `key >= start_key` and (if `end_key` is provided) `key < end_key`
3. If we exhaust the current leaf and `next_sibling != NO_SIBLING`, read the sibling page via `PageManager.read_page(next_sibling)` and repeat
4. Stop when `next_sibling == NO_SIBLING` (rightmost leaf reached) or a key exceeds `end_key`

The test at `test_btree.py:77-91` confirms both behaviors:
```python
# Bounded range: returns k05..k09 (excludes end_key k10)
results = tree.range_scan("k05", "k10")
keys = [k for k, v in results]
assert keys == ["k05", "k06", "k07", "k08", "k09"]

# Unbounded range: returns k15 through the last leaf
results = tree.range_scan("k15")
keys = [k for k, v in results]
assert keys == ["k15", "k16", "k17", "k18", "k19"]
```

With `max_keys_per_page=4` and 20 keys inserted, the tree must have at least 5 leaf pages. The unbounded scan starting at `k15` necessarily crosses leaf boundaries — it reads the leaf containing `k15`, follows `next_sibling` to the next leaf, and continues until `NO_SIBLING`.

## Why This Matters

Without sibling links, a range scan would need to either:
- Climb back up internal nodes to find the next leaf (expensive — potentially O(log n) per leaf transition)
- Re-traverse from the root for each successive leaf

The sibling chain gives O(1) leaf-to-leaf traversal, making range scans proportional to the number of results, not the tree height. This is the core design choice that makes B-trees practical for ordered range queries — and it's the same approach used by real database indexes (PostgreSQL's nbtree, InnoDB, etc.).

## What's Missing

The observations include only the first 200 of 612 lines of `btree.py`. The actual `range_scan` implementation at line 486, the `_search` method, and the leaf-splitting code (which must wire up `next_sibling` pointers when a leaf splits) are not visible. Reading these would complete the picture.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:range_scan` — The actual implementation at line 486; confirm the leaf-chain walk loop and how it handles the unbounded case
- [function] `b-tree-storage-engine/btree.py:_search` — The top-down traversal that finds the starting leaf for a range scan
- [general] `leaf-split-sibling-wiring` — When a leaf splits, the new leaf must be inserted into the sibling chain; understanding how `next_sibling` is maintained during splits is critical to correctness
- [function] `b-tree-storage-engine/btree.py:__iter__` — The full-tree iterator likely walks the entire leaf chain from the leftmost leaf, reusing the same `next_sibling` mechanism
- [file] `log-structured-merge-tree/lsm.py` — Compare with the LSM `range_scan` at line 272, which uses a fundamentally different approach (merge across sorted runs) instead of sibling pointers

## Beliefs

- `leaf-page-format-sibling-after-header` — Every leaf page stores a 4-byte `next_sibling` page number immediately after the 3-byte page header (type + num_keys), before any key-value entries
- `no-sibling-sentinel-is-max-uint32` — The sentinel `NO_SIBLING = 0xFFFFFFFF` marks the rightmost leaf in the chain; the range scan loop terminates when it encounters this value
- `range-scan-end-key-exclusive` — `range_scan("k05", "k10")` returns keys `k05` through `k09`, treating `end_key` as an exclusive upper bound
- `range-scan-supports-unbounded` — Calling `range_scan(start_key)` without an `end_key` scans from `start_key` through the last key in the rightmost leaf

