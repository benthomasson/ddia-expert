# Function: range_scan in b-tree-storage-engine/btree.py

**Date:** 2026-05-28
**Time:** 19:14

## `BTree.range_scan` — Leaf-chasing range query over a disk-backed B-tree

### Purpose

`range_scan` implements a half-open range query `[start_key, end_key)` over a sorted, disk-backed B-tree. It returns all key-value pairs whose keys fall within the specified range, in sorted order. This is the fundamental operation that makes B-trees useful for databases — unlike hash indexes, a B-tree maintains sort order across its leaves, enabling efficient retrieval of contiguous key ranges (e.g., "all orders between January and March").

### Contract

- **Precondition**: `start_key` must be a string (the tree stores UTF-8 encoded string keys). If `end_key` is provided, it must also be a string and should be `>= start_key` for meaningful results — but this isn't enforced; you'll just get an empty list.
- **Postcondition**: Returns a list of `(key, value)` tuples in sorted key order where `start_key <= key < end_key`. If `end_key is None`, returns everything from `start_key` to the maximum key.
- **Invariant**: The method is read-only with respect to tree structure. It never modifies pages or metadata. It does reset I/O counters as a side effect.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `start_key` | `str` | Inclusive lower bound of the range. The scan begins at the leaf that would contain this key. |
| `end_key` | `str` or `None` | Exclusive upper bound. `None` means "scan to the end of the tree." |

Edge cases:
- If `start_key` is greater than every key in the tree, an empty list is returned (the leaf is found but all keys are skipped, and there may be no further siblings).
- If `start_key == end_key`, the result is always empty (the bound is `<`, not `<=`).

### Return Value

`list[tuple[str, bytes]]` — a list of `(key, value)` pairs. The caller gets a materialized list, not a generator, so the entire result set lives in memory. For a large range, this could be substantial.

### Algorithm

1. **Reset I/O counters** — zeros out `pages_read` / `pages_written` on the `PageManager` so the caller can measure the I/O cost of this operation in isolation.

2. **Read metadata** — fetches root page number and tree height from page 0.

3. **Find the starting leaf** — calls `_find_leaf(root, start_key, height)`, which walks from root to leaf using `bisect_right` at each internal node. This takes exactly `height - 1` page reads for internal nodes plus landing on the leaf. The key insight: `bisect_right` finds the rightmost child whose key range could contain `start_key`.

4. **Linear scan across the leaf chain** — the core loop:
   - Read the current leaf page and deserialize it into `(keys, values, next_sibling)`.
   - For each `(k, v)` in the leaf:
     - **Skip** if `k < start_key` — this handles the first leaf, which may contain keys before our range.
     - **Early return** if `end_key is not None and k >= end_key` — we've passed the upper bound; return immediately.
     - Otherwise, **append** `(k, v)` to results.
   - Follow `next_sib` to the next leaf. If `next_sib == NO_SIBLING` (`0xFFFFFFFF`), we've hit the rightmost leaf and the loop ends.

5. **Return** the accumulated results list.

The scan is `O(height + R)` in page reads, where `R` is the number of leaf pages that overlap the range — optimal for a B-tree.

### Side Effects

- **Resets I/O counters**: `self.pm.reset_counters()` zeros `pages_read` and `pages_written`. This is a mutation that affects any concurrent reader relying on those counters (though this implementation is single-threaded).
- **Disk I/O**: Reads `height - 1` internal pages (via `_find_leaf`) plus however many leaf pages the range spans. Each `read_page` call increments `pages_read`.

### Error Handling

None explicitly. The method assumes:
- The tree file is well-formed (no corrupt pages).
- `_deserialize_leaf` won't fail on valid page data.
- The sibling chain terminates (no cycles). A cycle would cause an infinite loop — there's no visited-set guard.

If the tree file is truncated or corrupt, `struct.unpack` inside `_deserialize_leaf` would raise `struct.error`. This propagates uncaught to the caller.

### Usage Patterns

```python
tree = BTree('/tmp/mydb')
tree.put("user:001", b"Alice")
tree.put("user:002", b"Bob")
tree.put("user:003", b"Carol")

# Range query: all users
all_users = tree.range_scan("user:")  # end_key=None → scan to end

# Bounded range
some_users = tree.range_scan("user:001", "user:003")
# → [("user:001", b"Alice"), ("user:002", b"Bob")]  # excludes 003

# After the call, check I/O cost:
print(tree.pm.pages_read)  # number of pages touched
```

Caller obligations: the tree must not be modified during iteration (no concurrent `put`/`delete`). The result is a snapshot — later mutations don't affect the returned list.

### Dependencies

| Dependency | Role |
|------------|------|
| `PageManager.read_page` | Reads raw page bytes from the data file |
| `PageManager.read_meta` / `_read_meta` | Fetches root page and tree height |
| `_deserialize_leaf` | Parses raw bytes into `(keys, values, next_sibling)` |
| `_find_leaf` | Navigates from root to the target leaf using `bisect_right` at internal nodes |
| `_deserialize_internal` | Used internally by `_find_leaf` to parse internal nodes |

### Assumptions Not Enforced by the Type System

1. **Keys are strings** — the comparison `k < start_key` assumes both are the same type. Passing a non-string key would produce undefined comparison behavior.
2. **Leaf sibling chain is acyclic** — no cycle detection exists. A corrupt `next_sibling` pointer could loop forever.
3. **Keys within each leaf are sorted** — the `continue` / early-return logic depends on sort order. If a leaf's keys were out of order (due to a bug in `put`), results would be incorrect or incomplete.
4. **No concurrent modification** — reading pages while another thread writes could yield torn reads or inconsistent sibling pointers.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_find_leaf` — The tree-descent logic that locates the starting leaf; understanding `bisect_right` routing is essential to understanding why range scans are efficient.
- [function] `b-tree-storage-engine/btree.py:__iter__` — The full-tree scan variant that starts from the leftmost leaf rather than a specific key; compare to see how `range_scan` is a generalization.
- [function] `b-tree-storage-engine/btree.py:_insert` — How leaf splits maintain the sibling chain that `range_scan` follows; a split bug would break range scans.
- [general] `leaf-sibling-chains` — B+ tree leaf linking is a design choice not present in classic B-trees; explore why DDIA implementations use it and the tradeoff vs. re-descending from the root.
- [file] `log-structured-merge-tree/test_lsm.py` — Compare how LSM trees implement range scans (merging sorted runs) vs. this B-tree's leaf-chain walk.

---

## Beliefs

- `range-scan-half-open-interval` — `range_scan` returns keys in the half-open interval `[start_key, end_key)`; the start is inclusive and the end is exclusive.
- `range-scan-materializes-results` — Results are collected into a list in memory, not yielded lazily; large ranges consume proportional memory.
- `range-scan-follows-sibling-chain` — The scan walks the leaf-level linked list via `next_sibling` pointers rather than re-descending the tree for each leaf, making it O(height + leaf pages in range).
- `range-scan-resets-io-counters` — Each call to `range_scan` resets `pages_read` and `pages_written` to zero before starting, so counter values reflect only this operation's I/O.
- `range-scan-no-cycle-guard` — There is no protection against a corrupted sibling pointer creating a cycle; such corruption would cause an infinite loop.

