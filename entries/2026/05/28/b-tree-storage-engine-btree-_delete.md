# Function: _delete in b-tree-storage-engine/btree.py

**Date:** 2026-05-28
**Time:** 18:29

## `_delete` — Recursive B-Tree Key Deletion

### Purpose

`_delete` is the recursive engine behind `BTree.delete()`. It walks down the tree from a given node to find and remove a key from its leaf, then handles one specific structural cleanup on the way back up: removing emptied leaf nodes when they have a left sibling at depth 2. It exists to separate the tree-traversal logic from the metadata bookkeeping that the public `delete()` method handles (decrementing `total_keys`, committing the WAL).

### Contract

**Preconditions:**
- `page_num` refers to a valid, allocated page in the data file
- `depth` accurately reflects the distance from this node to the leaves (1 = leaf, 2 = parent of leaves, etc.)
- `depth` matches the subtree structure — calling with a wrong depth will misinterpret page contents

**Postconditions:**
- If the key existed: the key-value pair is removed from its leaf, all modified pages are written via WAL, and the return value is truthy
- If an emptied leaf is removed (depth 2, non-leftmost child): the previous sibling's `next_sibling` pointer is patched to maintain the linked-list scan order, and the empty page is returned to the free list
- The tree's structural invariants are maintained *except* that underflow/rebalancing is not performed — a leaf can end up with zero keys if it's the leftmost child or at depth > 2

**Invariants preserved:**
- Leaf linked-list order (via sibling pointer patching)
- Sorted key order within all nodes
- Internal node key/child count relationship (`len(children) == len(keys) + 1`)

### Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `page_num` | `int` | Page number of the current node in the data file |
| `key` | `str` | The key to delete (must match exactly via `==`) |
| `depth` | `int` | Height of the subtree rooted here. `1` = leaf, `2` = parent of leaves, etc. |

### Return Value

Returns a three-valued signal to the caller:

| Value | Meaning | Caller action |
|-------|---------|---------------|
| `False` | Key not found | Do nothing |
| `True` | Key deleted, leaf still has keys (or empty leaf was already cleaned up) | Propagate success |
| `'empty'` | Key deleted, leaf is now empty and was **not** cleaned up at this level | Parent should attempt cleanup |

The public `delete()` method treats any truthy return as success and decrements `total_keys`.

### Algorithm

**Leaf case (depth == 1):**

1. Read and deserialize the leaf page.
2. Binary-search for the key using `bisect_left`.
3. If the key isn't present, return `False`.
4. Remove the key and value at that index.
5. Serialize and write the modified leaf via WAL.
6. Return `'empty'` if the leaf has no remaining keys, otherwise `True`.

**Internal node case (depth > 1):**

1. Read and deserialize the internal node to get separator keys and child pointers.
2. Use `bisect_right` to find which child subtree contains the key.
3. Recurse into that child with `depth - 1`.
4. **Cleanup path** — if the child reported `'empty'`, `depth == 2` (this node is the direct parent of leaves), and `idx > 0` (the empty leaf is not the leftmost child):
   - Read the empty leaf to get its `next_sibling` pointer.
   - Read the previous sibling leaf.
   - Rewrite the previous sibling with the empty leaf's `next_sibling`, splicing the empty leaf out of the linked list.
   - Remove the separator key (`ikeys[idx-1]`) and the child pointer (`children[idx]`).
   - Free the empty page back to the page manager's free list.
   - Rewrite the current internal node.
   - Return `True` (cleanup done, don't propagate `'empty'` further).
5. If `result == 'empty'` but cleanup conditions aren't met (leftmost child, or depth > 2), return `True` — swallow the signal. The empty leaf stays allocated.
6. Otherwise, pass through the `False` or `True` result unchanged.

### Side Effects

- **Disk I/O**: Reads 1 page per level on the way down. On the cleanup path, reads 2 additional pages (empty leaf + previous sibling) and writes 3 pages (previous sibling, current internal node, plus the free-list write inside `free_page`).
- **WAL writes**: Every page mutation goes through `_wal_write_page`, which logs to the WAL file *and* writes to the data file. This is not yet committed — `delete()` calls `wal.commit()` after `_delete` returns.
- **Free list mutation**: `pm.free_page()` modifies the metadata page's free-list head pointer and writes a free-list node into the freed page. This bypasses the WAL (it calls `pm.write_page` directly, not `_wal_write_page`), which is a potential crash-safety gap.
- **I/O counters**: `pm.pages_read` and `pm.pages_written` are incremented by all page reads/writes.

### Error Handling

No explicit error handling. If the page data is corrupt or `page_num` is invalid, this will raise from `struct.unpack` or produce garbage. The method trusts that the tree structure on disk is consistent.

### Usage Patterns

Called exclusively by `BTree.delete()`:

```python
def delete(self, key):
    root, height, total_keys, next_free, free_head = self._read_meta()
    found = self._delete(root, key, height)
    if found:
        self._wal_write_meta(root, height, total_keys - 1, next_free, free_head)
        self.wal.commit(self.pm)
    return bool(found)
```

The caller is responsible for updating the key count and committing the WAL transaction. The `bool()` cast collapses `'empty'` and `True` into `True` for the public API.

### Dependencies

- `bisect_left`, `bisect_right` — standard library binary search
- `_deserialize_leaf`, `_serialize_leaf` — leaf page serialization
- `_deserialize_internal`, `_serialize_internal` — internal page serialization
- `self.pm` (`PageManager`) — page-level I/O, free-list management
- `self._wal_write_page` — WAL-logged page writes

### Notable Assumptions and Limitations

1. **No rebalancing.** This is a simplified B-tree that doesn't merge or redistribute keys on underflow. An emptied leftmost leaf (idx == 0) or an empty leaf at depth > 2 is left in place with zero keys — it wastes a page and stays in the tree permanently.

2. **Only cleans up at depth 2.** The `depth == 2` guard means cleanup only happens when the current node is the direct parent of leaves. If a deletion empties a leaf deeper in the tree, the `'empty'` signal is swallowed as `True` at the first internal node that can't handle it.

3. **`free_page` bypasses WAL.** When a leaf is freed during cleanup, `pm.free_page()` writes directly to the data file without WAL logging. If the process crashes after `free_page` but before `wal.commit()`, the free list and the internal node could be inconsistent.

4. **`bisect_right` vs `bisect_left`.** Leaves use `bisect_left` (find exact match position), internals use `bisect_right` (route to the correct child). This matches B-tree convention where a key equal to a separator routes right.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_insert` — The insertion counterpart handles splits; comparing the two reveals the asymmetry (splits are fully propagated, merges are not)
- [function] `b-tree-storage-engine/btree.py:range_scan` — Uses the leaf sibling linked list that `_delete` must maintain during cleanup
- [file] `b-tree-storage-engine/test_btree.py` — Test cases reveal which deletion edge cases (empty leftmost leaf, deep trees) are covered
- [general] `b-tree-underflow-rebalancing` — How production B-trees (SQLite, InnoDB) handle underflow with merge and redistribute operations, which this implementation omits
- [function] `b-tree-storage-engine/btree.py:WAL.recover` — Understanding WAL recovery clarifies why the `free_page` bypass is a crash-safety concern

## Beliefs

- `btree-delete-no-rebalance` — `_delete` never merges or redistributes keys between siblings; underflowing leaves persist with zero keys if they are the leftmost child or at depth > 2
- `btree-delete-cleanup-depth2-only` — Empty leaf cleanup only triggers when the current internal node is at depth 2 (direct parent of leaves) and the empty child is not the leftmost
- `btree-free-page-bypasses-wal` — `pm.free_page()` called during deletion writes directly to the data file without WAL logging, creating a potential crash-consistency gap
- `btree-delete-three-valued-return` — `_delete` uses a three-valued return (`False` / `True` / `'empty'`) to signal upward whether cleanup is needed, collapsed to `bool` at the public API boundary
- `btree-leaf-sibling-patched-on-remove` — When an empty leaf is removed, `_delete` patches the previous sibling's `next_sibling` pointer to maintain the linked list used by `range_scan` and `__iter__`

