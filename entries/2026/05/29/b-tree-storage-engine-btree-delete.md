# Function: delete in b-tree-storage-engine/btree.py

**Date:** 2026-05-29
**Time:** 07:38

## `BTree.delete(self, key)`

### Purpose

Removes a key-value pair from the disk-backed B-tree. This is the public API entry point for deletion — it coordinates the recursive tree traversal (`_delete`), metadata bookkeeping (decrementing `total_keys`), and WAL-based crash safety (atomic commit). Without this method, the B-tree would be append-only.

### Contract

- **Precondition**: `key` must be a string (the serialization layer in `_deserialize_leaf` calls `.decode('utf-8')` and comparisons assume string ordering). The tree must be in a consistent state (no partial WAL recovery pending — though the constructor handles recovery automatically).
- **Postcondition**: If the key existed, it is removed from the tree, `total_keys` is decremented by 1, and all changes are durably committed via WAL. If the key did not exist, the tree is unchanged and no WAL commit occurs.
- **Invariant**: The WAL commit only happens on successful deletion — a failed lookup produces no I/O beyond reads.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `str` | The key to delete. Must match the string encoding used during insertion. |

### Return Value

Returns `bool` — `True` if the key was found and deleted, `False` if the key was not present. The caller does not need to handle exceptions under normal operation; a `False` return is the "not found" signal.

### Algorithm

1. **Reset I/O counters** — `self.pm.reset_counters()` zeros the `pages_read`/`pages_written` counters so post-operation stats reflect only this deletion.

2. **Read metadata** — fetches the current `root` page number, tree `height`, `total_keys`, `next_free` page, and `free_head` (free-list pointer) from page 0.

3. **Recursive delete** — calls `_delete(root, key, height)`, which traverses the tree top-down:
   - At **leaf level** (`depth == 1`): binary-searches for the key, removes it if found, rewrites the leaf page through the WAL, and returns `'empty'` if the leaf is now keyless, `True` if deleted but keys remain, or `False` if not found.
   - At **internal levels**: routes to the correct child via `bisect_right`, recurses, and handles the case where a child leaf becomes empty — it patches the sibling linked list, removes the separator key, frees the empty page, and rewrites the internal node.

4. **Metadata update** — if `found` is truthy (`True` or `'empty'`), re-reads metadata to get the latest `next_free`/`free_head` (which may have changed if `_delete` freed a page), then writes updated metadata with `total_keys - 1` through the WAL.

5. **WAL commit** — `self.wal.commit(self.pm)` calls `fsync` on the data file, then truncates and `fsync`s the WAL. This is the atomic durability boundary — if a crash occurs before this point, WAL recovery replays the writes; after this point, the WAL is empty.

6. **Return** — `bool(found)` coerces `'empty'` to `True`.

### Side Effects

- **Disk I/O**: Reads and writes pages through `PageManager`. Writes are double-written (WAL first, then data file).
- **WAL writes**: Every modified page (leaf, internal, metadata, freed page) is logged before being applied.
- **Free list mutation**: If a leaf becomes empty at depth 2, `_delete` calls `pm.free_page()`, which modifies the free list by writing a free-list node and updating metadata. This is a **non-WAL-protected write** — `free_page` calls `pm.write_page` and `pm.write_meta` directly, bypassing the WAL.
- **Counter reset**: `pages_read`/`pages_written` are zeroed at entry.
- **fsync**: The WAL commit forces two fsyncs (data file + WAL file).

### Error Handling

No explicit exception handling. Potential failure modes:
- **I/O errors** from `read_page`/`write_page` propagate as `OSError`/`IOError` uncaught. If a crash occurs mid-operation, the WAL recovery on next startup replays logged writes — but note the free-list issue below.
- **Corrupt data** (e.g., truncated file) would cause `struct.unpack` to raise `struct.error`.
- The method **does not validate** that `key` is a string — passing bytes or other types would cause comparison failures or encoding errors deeper in the call stack.

### Critical Observation: Free-List / WAL Inconsistency

When `_delete` removes an empty leaf at depth 2, it calls `self.pm.free_page(child_page)`. This method writes directly to the data file via `pm.write_page` and `pm.write_meta` — **not** through `_wal_write_page`/`_wal_write_meta`. This means the free-list mutation is not crash-safe. A crash after `free_page` but before `wal.commit` could leave the free list in an inconsistent state while WAL recovery replays the old page contents. The metadata page gets overwritten twice in this path: once by `free_page` (unprotected) and once by `_wal_write_meta` (protected), so the WAL-protected version wins on recovery — but the freed page's free-list node content may be lost.

### Usage Patterns

```python
tree = BTree('/tmp/mydb')
tree.put('user:123', b'{"name": "Alice"}')

deleted = tree.delete('user:123')   # True
deleted = tree.delete('user:123')   # False — already gone

tree.close()
```

Callers should check the return value if they need to distinguish "key removed" from "key was never there." There is no `delete_if_exists` variant; the return value serves that purpose.

### Dependencies

| Dependency | Role |
|------------|------|
| `PageManager` | Fixed-size page I/O to the data file |
| `WAL` | Write-ahead log for atomic durability |
| `_deserialize_leaf`, `_serialize_leaf` | Leaf page serialization |
| `_deserialize_internal`, `_serialize_internal` | Internal page serialization |
| `bisect_left`, `bisect_right` | Binary search for key position |
| `struct` | Binary packing/unpacking of page headers and metadata |

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_delete` — The recursive core that handles leaf removal, sibling pointer patching, and internal node compaction — the actual tree surgery
- [function] `b-tree-storage-engine/btree.py:_insert` — Counterpart operation; understanding splits alongside merges gives the full picture of how the tree maintains balance
- [function] `b-tree-storage-engine/btree.py:WAL.commit` — The atomic durability boundary; understanding when fsync happens relative to WAL truncation is key to reasoning about crash safety
- [general] `btree-delete-underflow-handling` — This implementation only removes empty leaves at depth 2 and does not rebalance or merge underfull nodes, unlike textbook B-tree deletion — worth comparing with Knuth's algorithm or CLRS
- [file] `b-tree-storage-engine/test_btree.py` — Test coverage for delete edge cases: deleting the last key, deleting from internal nodes, and crash recovery scenarios

---

## Beliefs

- `btree-delete-no-rebalance` — Delete does not merge or redistribute underfull leaf nodes; it only removes completely empty leaves when their parent is at depth 2
- `btree-delete-wal-gap` — `pm.free_page()` called during deletion bypasses the WAL, creating a crash-safety gap in free-list management
- `btree-delete-metadata-reread` — Delete re-reads metadata after `_delete` returns to pick up free-list changes made by `free_page`, rather than threading updated values through the call stack
- `btree-delete-bool-coercion` — The `'empty'` sentinel from `_delete` is coerced to `True` via `bool(found)`, collapsing the internal "deleted and leaf is now empty" state before returning to the caller
- `btree-delete-no-commit-on-miss` — When the key is not found, no WAL entries are written and no commit occurs, making failed deletes read-only operations

