# Function: _insert in b-tree-storage-engine/btree.py

**Date:** 2026-05-28
**Time:** 19:13

## `BTree._insert` — Recursive B-Tree Insertion with Split Propagation

### Purpose

`_insert` is the recursive core of B-tree insertion. It walks down from the given node to the correct leaf, inserts the key-value pair, and handles node splits on the way back up via return values. The caller (`put`) uses the return value to decide whether to update metadata, grow the tree height, or do nothing.

This design separates the recursive tree-walking logic from the top-level bookkeeping (metadata updates, WAL commits, root splits), keeping the recursion focused on a single concern: "insert into this subtree and tell me what happened."

### Contract

**Preconditions:**
- `page_num` is a valid allocated page containing either a serialized leaf (if `depth == 1`) or a serialized internal node (if `depth > 1`)
- `depth` accurately reflects the distance from this node to the leaves (1 = leaf, 2 = parent of leaves, etc.)
- `key` is a string (it's used with `bisect_left`/`bisect_right` against deserialized UTF-8 strings)
- `value` is `bytes` (passed directly to `_serialize_leaf`, which writes it with a length prefix and no encoding)
- The key-value pair fits within a single page (enforced by `put`, not here)

**Postconditions:**
- If the key already existed, its value is overwritten in-place; no structural change occurs
- If the key is new and the target leaf has capacity, it's inserted in sorted position
- If insertion causes overflow, the node splits: the original page keeps the left half, a newly allocated page gets the right half, and the split key propagates upward
- All page writes go through `_wal_write_page`, so every mutation is WAL-logged before being applied

**Invariants maintained:**
- Keys within each node remain sorted
- Leaf sibling pointers form a valid linked list (right page inherits the old `next_sib`, left page points to the new right page)
- Internal nodes maintain the B-tree property: `len(children) == len(keys) + 1`

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `page_num` | `int` | Page number of the current node in the data file |
| `key` | `str` | The key to insert. Compared lexicographically via `bisect_left`/`bisect_right` |
| `value` | `bytes` | The value to store. Opaque to the tree; serialized as-is |
| `depth` | `int` | Height of the subtree rooted at this node. `1` means leaf; `>1` means internal. Decremented on each recursive call |

### Return Value

The return type is a three-way discriminated union — the caller must check the type to determine what happened:

| Return | Meaning | Caller obligation |
|--------|---------|-------------------|
| `None` | Key existed; value was updated in-place | No metadata change needed |
| `'inserted'` | New key inserted; no split | Increment `total_keys` in metadata |
| `(mid_key, new_page_num)` | Node split occurred | Insert `mid_key` into the parent, pointing to `new_page_num` as the new right child. If this is the root, create a new root |

### Algorithm

**Leaf case (`depth == 1`):**

1. Read and deserialize the page into `keys`, `values`, and `next_sib` (sibling pointer).
2. Binary search (`bisect_left`) to find the insertion point `idx`.
3. **Key exists** (`keys[idx] == key`): overwrite `values[idx]`, rewrite the page, return `None`.
4. **Key is new**: insert `key` and `value` at position `idx`.
5. **No overflow** (`len(keys) <= max_keys`): serialize and write the page, return `'inserted'`.
6. **Overflow — split**:
   - Split at `mid = len(keys) // 2`. Left half stays in the original page; right half goes to a new page.
   - The split key (`mid_key`) is `right_keys[0]` — the first key of the right page. Note: in leaf splits, the split key is **copied** up (it remains in the right leaf), unlike internal splits where it's **promoted** out.
   - The right page inherits the old `next_sib` pointer (preserving the leaf chain).
   - The left page's `next_sib` is set to the new right page number.
   - Return `(mid_key, new_page)`.

**Internal case (`depth > 1`):**

1. Deserialize the internal node into `ikeys` and `children`.
2. Use `bisect_right` to find which child subtree the key belongs in. (`bisect_right` ensures equal keys go to the right child, which is the standard B-tree convention for internal routing.)
3. **Recurse** into `children[idx]` with `depth - 1`.
4. **No split from child** (`None` or `'inserted'`): propagate the result unchanged.
5. **Child split**: the child returned `(mid_key, new_child)`.
   - Insert `mid_key` at position `idx` in `ikeys`.
   - Insert `new_child` at position `idx + 1` in `children` (the new right child goes immediately after the old child that split).
6. **No overflow**: serialize and write the updated internal node, return `'inserted'`.
7. **Overflow — split internal**:
   - Split at `mid = len(ikeys) // 2`. The key at position `mid` is **promoted** — it goes to the parent and does **not** appear in either the left or right child.
   - Left: `ikeys[:mid]` with `children[:mid + 1]`.
   - Right: `ikeys[mid + 1:]` with `children[mid + 1:]`.
   - Return `(promote_key, new_page)`.

The difference between leaf and internal splits matters: leaves **copy** `mid_key` (it stays in the right leaf for lookups), while internal nodes **promote** it (it's removed from both children).

### Side Effects

- **Disk I/O**: Every page mutation calls `_wal_write_page`, which writes to the WAL file (with `fsync`) and then writes to the data file. A single insert touching a leaf can write 1 page (update/simple insert), or 2-3 pages (split), plus additional pages if the split cascades up.
- **Page allocation**: Splits call `self.pm.allocate_page()`, which modifies the metadata page (incrementing `next_free` or popping from the free list). This metadata write happens via `PageManager.write_meta`, which is **not** WAL-logged from within `_insert` — only the caller (`put`) WAL-logs the final metadata update.
- **No WAL commit**: `_insert` never calls `wal.commit()`. The caller (`put`) is responsible for committing after all pages and metadata are written. This means all writes within a single `put` are atomic with respect to crash recovery.

### Error Handling

`_insert` does not catch any exceptions. Possible failures:

- **I/O errors** from `pm.read_page` or `pm.write_page` propagate as `OSError`/`IOError`
- **Deserialization errors** (corrupt pages) would raise `struct.error` or index errors
- **Page too large**: if serialized data exceeds `page_size`, `write_page` silently truncates it (`data[:self.page_size]`), which would corrupt the page — this is a latent bug, though `put` guards against oversized single entries

There's no rollback on partial failure. If the process crashes mid-insert (after writing some pages but before WAL commit), recovery replays the WAL, which re-applies all logged writes. But if `allocate_page` updated metadata before the WAL was committed, the metadata page could be inconsistent — this is a known gap in the WAL coverage within `_insert`.

### Usage Patterns

`_insert` is called exactly once, from `put`:

```python
result = self._insert(root, key, value, height)
```

The caller then dispatches on the return value:
- `None` → commit WAL, done
- `'inserted'` → re-read metadata, increment `total_keys`, WAL-write metadata, commit
- `(mid_key, new_page)` → root split: allocate a new root page, create an internal node with the old root and new page as children, update metadata with the new root and incremented height, commit

Callers must never call `_insert` without subsequently committing the WAL — leaving uncommitted WAL entries would cause them to be replayed on next startup.

### Dependencies

- `PageManager` (`self.pm`) — page-level read/write/allocate
- `WAL` (via `self._wal_write_page`) — crash-safety logging
- `_serialize_leaf` / `_deserialize_leaf` — leaf page encoding
- `_serialize_internal` / `_deserialize_internal` — internal page encoding
- `bisect_left` / `bisect_right` from stdlib — binary search over sorted key arrays
- `self.max_keys` — branching factor; determines when nodes overflow

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:put` — The public entry point that calls `_insert`, handles root splits, metadata updates, and WAL commits
- [function] `b-tree-storage-engine/btree.py:_delete` — Deletion logic; notably simpler (no merge/rebalance, only handles empty-leaf cleanup), interesting to contrast with the split-heavy insertion
- [general] `wal-atomicity-gap` — `allocate_page` writes metadata outside the WAL boundary; worth tracing whether this creates a crash-safety hole
- [function] `b-tree-storage-engine/btree.py:range_scan` — Uses the leaf sibling chain that `_insert` maintains during splits; good test of whether the linked list invariant holds
- [file] `b-tree-storage-engine/test_btree.py` — Tests covering split cascades, crash recovery, and range scans that exercise the paths through `_insert`

---

## Beliefs

- `btree-insert-three-way-return` — `_insert` returns `None` (update), `'inserted'` (no split), or a `(key, page)` tuple (split occurred); callers must handle all three
- `btree-leaf-split-copies-key` — Leaf splits copy the midpoint key into the right leaf (it appears in both the parent and the right leaf), while internal splits promote the key out of both children
- `btree-insert-no-wal-commit` — `_insert` never commits the WAL; the caller (`put`) is solely responsible for calling `wal.commit` after all page writes complete
- `btree-sibling-chain-maintained` — During leaf splits, the right page inherits the old `next_sib` and the left page points to the new right page, preserving the leaf linked list for range scans
- `btree-allocate-page-outside-wal` — Page allocation during splits modifies metadata through `PageManager.write_meta` (not WAL-logged), creating a potential crash-safety gap if the process fails between allocation and WAL commit

