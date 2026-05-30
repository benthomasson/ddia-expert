# Topic: DDIA Chapter 3 covers why B+-trees link leaves in a chain (vs. B-trees which store values at every level), and the tradeoffs for sequential scan performance

**Date:** 2026-05-29
**Time:** 10:38

# B+-Tree Leaf Chains and Sequential Scan Performance

## The Core Design Decision

This implementation is a **B+-tree**, not a plain B-tree. The distinction is visible right in the on-disk format. Look at `_serialize_leaf` at `b-tree-storage-engine/btree.py:178`:

```python
def _serialize_leaf(keys, values, next_sibling=NO_SIBLING):
    buf = struct.pack(HEADER_FMT, LEAF, len(keys))
    buf += struct.pack('>I', next_sibling)
    for k, v in zip(keys, values):
        ...
```

Every leaf node carries two things a plain B-tree internal node wouldn't: **the actual values** (not just routing keys) and a **`next_sibling` pointer** (a 4-byte page number). The sentinel `NO_SIBLING = 0xFFFFFFFF` at line 14 marks the end of the chain.

Internal nodes, by contrast, store only keys and child page pointers — no values. This is the defining split between B-trees and B+-trees: **values live exclusively in leaves**.

## Why Link Leaves?

In a plain B-tree, if you want to scan keys `"cherry"` through `"grape"`, you'd do an in-order traversal of the tree — descending into a subtree, visiting a key in an internal node, descending into the next subtree, back up, and so on. Every key visit potentially requires navigating up and down the tree. For a range scan of *N* results in a tree of height *h*, you might touch *O(N × h)* pages.

The B+-tree leaf chain converts this into a **linked-list walk**. Once you've descended to the first matching leaf (costing *h* page reads), you follow `next_sibling` pointers horizontally. Each subsequent leaf is one page read. For *N* results spread across *L* leaves, the cost is *h + L* page reads — and the *L* reads are **sequential** on disk if leaves were allocated in order.

The test at `b-tree-storage-engine/test_btree.py:82` exercises this directly:

```python
results = tree.range_scan("k05", "k10")
keys = [k for k, v in results]
assert keys == ["k05", "k06", "k07", "k08", "k09"]
```

This scan spans multiple leaf pages (with `max_keys_per_page=4`, 20 keys require at least 5 leaves). The `range_scan` method descends to `"k05"`, then walks the sibling chain until it hits `"k10"` — no re-traversal of internal nodes.

## The Leaf Format on Disk

The binary layout makes the chain explicit. From `_write_empty_leaf` at line 52:

```python
def _write_empty_leaf(self, page_num):
    data = struct.pack(HEADER_FMT, LEAF, 0)
    data += struct.pack('>I', NO_SIBLING)  # next sibling
```

Each leaf page is structured as:

| Field | Size | Purpose |
|-------|------|---------|
| Page type | 1B | `LEAF=1` vs `INTERNAL=0` |
| Key count | 2H | Number of entries |
| Next sibling | 4B | Page number of right neighbor |
| Key-value pairs | variable | Length-prefixed keys and values |

The `next_sibling` field sits right after the header — it's the first thing read after determining this is a leaf. `_deserialize_leaf` (line 189) returns it as the third element of the tuple: `(keys, values, next_sibling)`.

## Tradeoffs

**What the chain buys you:**
- Range scans and full iteration are *O(h + L)* instead of *O(N × h)* — the dominant use case for database queries with `WHERE key BETWEEN x AND y`.
- Sequential I/O patterns: adjacent leaves can be prefetched by the OS because the scan direction is predictable.
- The full-table scan in `test_basic` (line 33: `all_keys = [k for k, v in tree]`) walks the entire leaf chain without touching a single internal node after the initial descent.

**What it costs:**
- 4 bytes per leaf for the `next_sibling` pointer — trivial overhead per page.
- Splits must update the chain: when a leaf splits, the new leaf inherits the old leaf's `next_sibling`, and the old leaf's `next_sibling` is rewritten to point to the new leaf. This is one extra page write per split.
- Point lookups pay a minor penalty: since values aren't stored in internal nodes, every lookup must descend all the way to a leaf — you can never "short-circuit" by finding the key at an internal level. The `test_page_io_counts` test at line 99 validates this: reads should equal `height + 1` at most.

**The DDIA insight:** For workloads dominated by range queries (which most OLTP databases are), the leaf chain's sequential scan advantage far outweighs the extra descent cost on point lookups. This is why virtually every production database uses B+-trees, not B-trees.

## What's Missing From These Observations

The observations only include the first 200 lines of `btree.py` (612 total). To fully trace the leaf chain mechanics, you'd want to read:
- The `range_scan` method — to see how it follows `next_sibling` pointers
- The split logic — to see how the chain is maintained when a leaf overflows
- The `__iter__` method — to see full-tree iteration via the leaf chain
- `_serialize_internal` / `_deserialize_internal` — to confirm internal nodes store only keys + child pointers, no values

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:range_scan` — How the leaf sibling chain is actually walked during range queries; this is where the O(h + L) cost manifests
- [function] `b-tree-storage-engine/btree.py:_serialize_internal` — Confirm internal nodes store only routing keys and child pointers, completing the B+-tree picture
- [general] `leaf-split-chain-maintenance` — When a leaf overflows and splits, how the sibling pointers are rewired; a bug here silently breaks range scans
- [function] `b-tree-storage-engine/btree.py:__iter__` — Full-table scan implementation; likely descends to the leftmost leaf then walks the chain to the end
- [general] `sequential-io-and-page-allocation` — Whether `allocate_page` (line 93) tends to produce physically contiguous leaves, which determines whether the logical chain translates to actual sequential disk I/O

## Beliefs

- `leaf-nodes-store-all-values` — Values are stored exclusively in leaf nodes; internal nodes contain only routing keys and child page pointers, making this a B+-tree
- `leaf-sibling-chain-exists` — Every leaf carries a 4-byte `next_sibling` page pointer (sentinel `0xFFFFFFFF` for the rightmost leaf), forming a singly-linked chain across all leaves
- `range-scan-uses-leaf-chain` — `range_scan` descends to the start key's leaf then follows sibling pointers horizontally, never re-traversing internal nodes
- `point-lookup-always-reaches-leaf` — Every `get` must descend to a leaf node regardless of tree height, since internal nodes hold no values; cost is exactly *h* page reads for a tree of height *h*

