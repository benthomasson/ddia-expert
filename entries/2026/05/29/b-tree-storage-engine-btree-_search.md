# Function: _search in b-tree-storage-engine/btree.py

**Date:** 2026-05-29
**Time:** 10:35

## `BTree._search` — Point lookup traversal from root to leaf

### Purpose

`_search` is the recursive core of B-tree point lookups. It traverses from an internal node down to the leaf level, reading one page per level, and returns the value associated with a key or `None` if the key doesn't exist. It exists to separate the traversal logic from the public `get()` method, which handles metadata reads and counter resets.

### Contract

**Preconditions:**
- `page_num` refers to a valid, allocated page containing either a serialized leaf (when `depth == 1`) or a serialized internal node (when `depth > 1`)
- `depth` accurately reflects the distance from this node to the leaf level — the tree is perfectly balanced, so every path from root to leaf has the same depth
- `key` is a string (the serialization functions decode keys as UTF-8 strings)

**Postconditions:**
- Returns `bytes` (the stored value) if the key exists, `None` otherwise
- The tree is unmodified — this is a read-only operation

**Invariant assumed:** The tree is balanced. `depth` is decremented by exactly 1 at each recursive call, and `depth == 1` always means "this page is a leaf." If the metadata height is wrong (e.g., after a corrupted write), the method will deserialize a page using the wrong format and produce garbage or crash.

### Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `page_num` | `int` | Page number to read from the data file. Multiplied by `page_size` to get the byte offset. |
| `key` | `str` | The lookup key. Must be a string — `bisect_left`/`bisect_right` compare it against deserialized string keys. |
| `depth` | `int` | Levels remaining to the leaf. `1` = this page is a leaf; `>1` = internal node. Caller sets this to the tree's height from metadata. |

**Edge cases:** If `depth` is 0 or negative, the method falls through to the internal-node branch and tries to deserialize whatever is at `page_num` as an internal node — no guard exists. This can't happen in normal operation because `get()` reads the height from metadata (minimum 1).

### Return Value

- **`bytes`** — the value associated with the key, if found in the leaf
- **`None`** — the key does not exist in the subtree

The caller (`get()`) passes this through directly, so the public API also returns `bytes | None`.

### Algorithm

1. **Read the page** — `self.pm.read_page(page_num)` fetches `page_size` bytes from offset `page_num * page_size` in the data file. This increments `pages_read`.

2. **Leaf case (`depth == 1`)**:
   - Deserialize the page as a leaf, extracting sorted `keys`, `values`, and a `next_sibling` pointer (discarded here — only `range_scan` follows sibling chains).
   - Use `bisect_left(keys, key)` to find the insertion point. `bisect_left` is correct here because we want the *leftmost* position where `key` could appear — if `keys[idx] == key`, that's our match.
   - Check `idx < len(keys) and keys[idx] == key`. Both conditions are necessary: `bisect_left` can return `len(keys)` if `key` is larger than all stored keys.
   - Return the value at that index, or `None`.

3. **Internal case (`depth > 1`)**:
   - Deserialize as an internal node, extracting separator `ikeys` and `children` pointers.
   - Use `bisect_right(ikeys, key)` to pick the child. `bisect_right` is the correct choice: for an internal node with keys `[k0, k1, ...]` and children `[c0, c1, c2, ...]`, child `c_i` covers keys in the range `[k_{i-1}, k_i)`. `bisect_right` ensures that a key equal to a separator goes to the *right* child, matching the split logic in `_insert` where the right page keeps the median key.
   - Recurse into `children[idx]` with `depth - 1`.

**Total I/O:** Exactly `height` page reads — one per tree level. For a tree with N keys and branching factor B, that's O(log_B(N)).

### Side Effects

- **Increments `self.pm.pages_read`** by 1 for each level traversed. The caller `get()` calls `reset_counters()` before the search, so after `get()` returns, `pm.pages_read` equals the tree height.
- **File I/O:** Reads from the data file via `self.pm._f.seek()` and `self.pm._f.read()`. No writes.

### Error Handling

None explicit. Possible failures:
- **`struct.unpack` errors** if the page data is corrupted or truncated — these propagate as `struct.error`
- **`IndexError`** if `children[idx]` is out of bounds — would indicate a corrupted internal node where the children array doesn't have `len(keys) + 1` entries
- **I/O errors** from `read_page` — propagate as `OSError`

All errors are uncaught and bubble to the caller. This is appropriate — there's no meaningful recovery from a corrupted page mid-search.

### Usage Patterns

Only called from `get()`:

```python
def get(self, key):
    self.pm.reset_counters()
    root, height, _, _, _ = self._read_meta()
    return self._search(root, key, height)
```

The caller reads metadata to obtain the root page and height, resets I/O counters, then delegates. No other method calls `_search` — `_find_leaf` is a separate traversal used by `range_scan` that only locates the leaf page number without checking for a specific key.

### Dependencies

| Dependency | Role |
|------------|------|
| `self.pm.read_page()` | Page-level I/O from `PageManager` |
| `_deserialize_leaf()` | Unpacks leaf binary format → `(keys, values, next_sibling)` |
| `_deserialize_internal()` | Unpacks internal binary format → `(keys, children)` |
| `bisect_left` / `bisect_right` | stdlib binary search from `bisect` module |

### Key Design Choice: `bisect_left` vs `bisect_right`

The method uses `bisect_left` for leaves and `bisect_right` for internal nodes. This is intentional and matches the split strategy in `_insert`: when a leaf splits, the median key is *copied up* to the parent and also stays in the right leaf as its first key. So `bisect_right` at internal nodes routes an exact-match key to the right child (where that key lives), and `bisect_left` at the leaf finds the exact position.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_insert` — The write-path counterpart; understanding how splits propagate explains why `bisect_right` is used for internal nodes
- [function] `b-tree-storage-engine/btree.py:_find_leaf` — Similar traversal but returns the leaf page number instead of a value; used by `range_scan` to start sequential leaf reads
- [function] `b-tree-storage-engine/btree.py:_deserialize_leaf` — The binary layout determines what `_search` actually gets back; understanding the format clarifies why keys are always strings
- [general] `bisect-left-vs-right-in-btrees` — The asymmetry between `bisect_left` (leaf) and `bisect_right` (internal) is a subtle but critical invariant tied to the split/promote strategy
- [file] `b-tree-storage-engine/test_btree.py` — Tests exercise get/put/delete/range_scan paths and demonstrate the expected behavior at tree boundaries

## Beliefs

- `btree-search-reads-exactly-height-pages` — `_search` performs exactly one `read_page` call per tree level, so a lookup in a tree of height H always reads H pages
- `btree-bisect-left-leaf-bisect-right-internal` — Leaf lookups use `bisect_left` for exact match; internal routing uses `bisect_right` to send equal keys to the right child, matching the split convention where the median key is kept in the right leaf
- `btree-search-is-read-only` — `_search` never writes pages, modifies metadata, or touches the WAL; it only calls `read_page`
- `btree-depth-param-not-validated` — `_search` trusts that `depth` correctly reflects the tree's balance; if metadata is corrupted and height is wrong, the method silently deserializes pages with the wrong format

