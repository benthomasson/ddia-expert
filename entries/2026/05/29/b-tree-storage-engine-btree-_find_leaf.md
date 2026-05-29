# Function: _find_leaf in b-tree-storage-engine/btree.py

**Date:** 2026-05-29
**Time:** 07:43

## `_find_leaf` — Leaf Page Locator

### Purpose

`_find_leaf` traverses the B-tree from a given internal node down to the leaf level, returning the **page number** of the leaf page that would contain the given key. It does not check whether the key actually exists — it finds where the key *would* live.

This exists to support `range_scan`, which needs to start iterating from a specific leaf page and then follow sibling pointers. Unlike `_search` (which reads the leaf and returns a value), `_find_leaf` stops one step earlier: it returns the leaf's page number so the caller can manage the leaf-level scan itself.

### Contract

**Preconditions:**
- `page_num` must be a valid allocated page number pointing to a node at the given `depth` in the tree.
- `depth >= 1`. At depth 1, the page is a leaf. At depth > 1, the page is an internal node.
- The tree structure must be consistent: internal nodes at depth `d` have children at depth `d-1`.
- `key` must be comparable with the keys stored in internal nodes (string-to-string ordering via `bisect_right`).

**Postcondition:** Returns the page number of the leaf page whose key range includes `key` — specifically, the leaf reachable by following the same child-pointer path that `_search` would take.

**Invariant:** Each recursive call decrements `depth` by exactly 1, guaranteeing termination in `depth - 1` recursive steps.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `page_num` | `int` | Page number of the current node being examined. On the initial call, this is the root page. |
| `key` | `str` | The search key. Used to choose which child pointer to follow at each internal level. |
| `depth` | `int` | Remaining depth from `page_num` to the leaf level. `1` means `page_num` is itself a leaf. |

**Edge cases:**
- `depth == 1` on a single-level tree (root is a leaf): returns `page_num` immediately with no I/O.
- `key` less than all internal keys: `bisect_right` returns 0, following `children[0]` (leftmost child).
- `key` greater than all internal keys: `bisect_right` returns `len(ikeys)`, following the rightmost child.

### Return Value

Returns an `int` — the page number of the leaf that would contain `key`. The caller (`range_scan`) then reads this page directly and begins scanning.

There is no "not found" return. The method always succeeds if the tree is well-formed.

### Algorithm

1. **Base case** (`depth == 1`): The current page is a leaf. Return its page number immediately.
2. **Recursive case** (`depth > 1`):
   - Read the page at `page_num` from disk via `self.pm.read_page`.
   - Deserialize it as an internal node, extracting separator keys (`ikeys`) and child page pointers (`children`). An internal node with *n* keys has *n+1* children.
   - Use `bisect_right(ikeys, key)` to find the insertion point. `bisect_right` returns the index of the first key strictly greater than `key`, which is exactly the index of the correct child pointer. This mirrors the B-tree search invariant: child at index `i` covers keys in the range `[ikeys[i-1], ikeys[i])`.
   - Recurse into `children[idx]` with `depth - 1`.

The choice of `bisect_right` (not `bisect_left`) matters: for a key equal to a separator, `bisect_right` directs to the **right** child, which is consistent with how `_insert` and `_search` route — the separator key belongs to the right subtree (visible in the leaf split logic where `mid_key = right_keys[0]`).

### Side Effects

- **Disk I/O**: Reads one page per internal level traversed (`depth - 1` reads total). These reads increment `self.pm.pages_read`.
- **No writes**: This is a read-only traversal.
- **No WAL interaction**.

### Error Handling

None. The method assumes:
- `page_num` is valid (no bounds checking on the file).
- The page at `page_num` is actually an internal node when `depth > 1` (no type check — `_deserialize_internal` will silently misparse a leaf page).
- The tree height is accurate (a stale `depth` value could cause it to stop too early or read past leaves).

A corrupted page or incorrect depth would produce silently wrong results rather than raising an exception.

### Usage Patterns

Called exclusively by `range_scan`:

```python
leaf_num = self._find_leaf(root, start_key, height)
```

The caller then reads that leaf, scans its keys, and follows `next_sib` pointers to walk through subsequent leaves. The caller is responsible for:
- Calling `self._read_meta()` to get the current root and height.
- Filtering keys in the returned leaf that are below `start_key` (since `_find_leaf` returns the *containing* leaf, not a precise key match).

### Dependencies

- **`self.pm.read_page(page_num)`** — `PageManager` method for fixed-size page reads from the data file.
- **`_deserialize_internal(data)`** — Module-level function that parses the binary page format into `(keys, children)`.
- **`bisect_right`** from Python's `bisect` module — binary search for the insertion point in sorted keys.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_search` — Compare with `_find_leaf`: same traversal logic but reads the leaf and returns a value rather than a page number
- [function] `b-tree-storage-engine/btree.py:range_scan` — The sole caller; understanding how it uses the returned leaf page and walks sibling pointers completes the picture
- [function] `b-tree-storage-engine/btree.py:_deserialize_internal` — The binary format that `_find_leaf` relies on; understanding the `[child0, key0, child1, key1, child2, ...]` layout clarifies why `children[idx]` is correct
- [general] `bisect-right-vs-left-in-btrees` — Why `bisect_right` is the correct choice here and how it interacts with the split strategy (separator = first key of right sibling)
- [function] `b-tree-storage-engine/btree.py:_insert` — The split logic that establishes the invariant `_find_leaf` depends on: separator keys route equal keys to the right child

## Beliefs

- `find-leaf-read-count` — `_find_leaf` performs exactly `height - 1` page reads to traverse from root to leaf, making its I/O cost proportional to tree height
- `find-leaf-bisect-right-consistency` — `_find_leaf` uses `bisect_right` (not `bisect_left`), matching the routing logic in `_search` and `_insert`, so all three methods agree on which leaf owns a given key
- `find-leaf-no-type-check` — `_find_leaf` does not verify that a page is actually an internal node before deserializing it; a corrupted tree height or wrong page type produces silent wrong results
- `find-leaf-range-scan-only` — `_find_leaf` is called exclusively by `range_scan`; point lookups use `_search` instead, which reads the leaf inline rather than returning its page number

