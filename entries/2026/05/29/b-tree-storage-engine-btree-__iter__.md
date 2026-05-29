# Function: __iter__ in b-tree-storage-engine/btree.py

**Date:** 2026-05-29
**Time:** 07:43

## `BTree.__iter__` — Full In-Order Traversal via Leaf Chain

### 1. Purpose

`__iter__` makes `BTree` instances iterable, letting you write `for key, val in tree: ...` or `list(tree)`. It yields every `(key, value)` pair stored in the tree in sorted key order. This is the full-table-scan equivalent for the B-tree — useful for serialization, debugging, or bulk reads where you need every record.

### 2. Contract

- **Precondition**: The tree must be in a consistent state (no concurrent writes during iteration). The metadata page, internal pages, and leaf sibling chain must be structurally valid.
- **Postcondition**: Every key-value pair is yielded exactly once, in ascending key order.
- **Invariant relied upon**: Leaf pages are linked left-to-right via `next_sibling` pointers, and the leftmost leaf is reachable by always following `children[0]` from the root down through every internal level.

### 3. Parameters

None beyond `self`. This is a generator method invoked implicitly by Python's iteration protocol.

### 4. Return Value

A **generator** yielding `(str, bytes)` tuples — the key (decoded UTF-8 string) and value (raw bytes) for every entry. When the tree is empty, the generator yields nothing. The caller receives pairs in sorted order and can stop early (the generator is lazy).

### 5. Algorithm

Two phases:

**Phase 1 — Descend to the leftmost leaf:**

```
page_num = root
for each internal level (height - 1 times):
    read the internal page
    follow children[0]   ← always the leftmost child pointer
```

This walks the left spine of the tree. If `height == 1`, the root is already a leaf and this loop is skipped entirely.

**Phase 2 — Scan the leaf chain:**

```
while page_num != NO_SIBLING (0xFFFFFFFF):
    deserialize the leaf at page_num
    yield each (key, value) pair
    advance to next_sibling
```

Each leaf stores a `next_sibling` pointer set during splits (see `_insert`). The chain terminates when `next_sibling == NO_SIBLING`. This is a classic B+-tree leaf scan — no backtracking up the tree is needed.

### 6. Side Effects

- **I/O**: Reads every internal page on the left spine (one per level above leaves) plus every leaf page. For a tree with `L` leaves and height `H`, that's `H - 1 + L` page reads.
- **Counter mutation**: Each `read_page` call increments `self.pm.pages_read`. Unlike `get`/`put`, `__iter__` does **not** call `reset_counters()` first, so the read count accumulates on top of whatever was already there.
- **No writes**: Purely read-only.

### 7. Error Handling

None. The method assumes all pages are readable and well-formed. A corrupted page, a truncated file, or an invalid sibling pointer would raise a `struct.unpack` error or silently produce garbage. There's no validation of page types during traversal — it trusts that the left-spine pages are internal and the bottom level is leaves.

### 8. Usage Patterns

```python
# Dump entire tree
for key, value in tree:
    print(key, value)

# Materialize to list
all_pairs = list(tree)

# Check if tree contains a value (linear scan — prefer tree.get() for single lookups)
found = any(v == target for _, v in tree)
```

**Caller obligation**: Do not mutate the tree during iteration. The generator holds a `page_num` cursor into the leaf chain; inserts that trigger splits could corrupt the chain mid-scan. There is no snapshot isolation or MVCC here.

### 9. Dependencies

| Dependency | Role |
|---|---|
| `self._read_meta()` | Gets root page number and tree height |
| `self.pm.read_page(n)` | Raw page I/O from `PageManager` |
| `_deserialize_internal(data)` | Parses internal node to extract `children[0]` |
| `_deserialize_leaf(data)` | Parses leaf node to extract keys, values, and next sibling pointer |
| `NO_SIBLING` (0xFFFFFFFF) | Sentinel marking the end of the leaf chain |

### Unenforceable Assumptions

- **Leaf chain completeness**: Assumes every leaf is reachable via the sibling chain starting from the leftmost leaf. If a split wrote the new right page but crashed before updating the left page's `next_sibling`, entries on the orphaned page would be silently skipped.
- **No concurrent modification**: No locking or snapshot mechanism prevents a `put()` from splitting a leaf that iteration is about to read.
- **`reset_counters` not called**: Unlike `get`, `put`, `range_scan`, and `delete`, this method doesn't reset I/O counters. If you're benchmarking page reads for iteration, you need to call `self.pm.reset_counters()` yourself beforehand.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:range_scan` — Uses `_find_leaf` instead of left-spine descent; compare the two leaf-scanning strategies and when each is appropriate
- [function] `b-tree-storage-engine/btree.py:_insert` — Where the `next_sibling` pointers that `__iter__` relies on are created during leaf splits
- [function] `b-tree-storage-engine/btree.py:_delete` — How deletion updates sibling pointers when merging empty leaves, and what happens if the chain breaks
- [general] `b-plus-tree-leaf-chains` — DDIA Chapter 3 covers why B+-trees link leaves in a chain (vs. B-trees which store values at every level), and the tradeoffs for sequential scan performance
- [file] `b-tree-storage-engine/test_btree.py` — Check whether iteration after deletes, splits, and crash recovery is covered by tests

## Beliefs

- `iter-yields-sorted-pairs` — `__iter__` yields `(key, value)` pairs in ascending key order by scanning the leaf sibling chain left-to-right, never re-traversing internal nodes
- `iter-no-counter-reset` — Unlike `get`, `put`, `delete`, and `range_scan`, `__iter__` does not call `reset_counters()`, so I/O stats accumulate across calls
- `leaf-chain-invariant` — The entire iteration depends on the invariant that leaf `next_sibling` pointers form a complete left-to-right chain; a broken link silently truncates results
- `iter-not-mutation-safe` — No concurrency control exists; mutating the tree during iteration can cause missed entries or struct deserialization errors if a split reallocates the page being read

