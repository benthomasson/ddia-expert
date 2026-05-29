# Function: free_page in b-tree-storage-engine/btree.py

**Date:** 2026-05-29
**Time:** 07:50

## `PageManager.free_page`

### Purpose

`free_page` returns a disk page to a **singly-linked free list** so it can be reused by future `allocate_page` calls. Without it, deleted B-tree nodes would leave permanent holes in the data file — the file would grow monotonically even under heavy delete workloads. This is the deallocation half of the page allocator; `allocate_page` is the allocation half.

### Contract

**Preconditions:**
- `page_num` must refer to a valid, currently-in-use page that is no longer referenced by any B-tree node. The caller is responsible for unlinking it from the tree first. Freeing a page that is still pointed to by an internal node or sibling pointer corrupts the tree silently — nothing enforces this.
- The metadata page (page 0) must be intact and readable.

**Postconditions:**
- The freed page becomes the new head of the free list.
- The old free-list head is stored inside the freed page's body, preserving the chain.
- `next_free` (the high-water mark for fresh page allocation) is unchanged — only the free-list head pointer in metadata is updated.

**Invariant preserved:** The free list is a singly-linked list threaded through the pages themselves, terminated by `NO_SIBLING` (`0xFFFFFFFF`). After this call, `free_head → page_num → old_free_head → ... → NO_SIBLING`.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `self` | `PageManager` | The page manager with an open data file |
| `page_num` | `int` | The page number to free. Must be a valid, unreferenced page. No bounds checking is performed. |

### Return Value

`None`. All effects are through side effects on disk.

### Algorithm

1. **Read current metadata** — fetches `(root, height, total_keys, next_free, free_head)` from page 0.
2. **Build a free-list node** — constructs a minimal page payload:
   - A zeroed-out header (`struct.pack(HEADER_FMT, 0, 0)` → page type `0`, key count `0`) — type 0 is `INTERNAL`, but with 0 keys it's recognizable as a free-list node.
   - A 4-byte big-endian pointer to the **old** free-list head, placed immediately after the header at offset `HEADER_SIZE`. This is the "next" pointer in the linked list.
3. **Write the free-list node** to `page_num` — overwrites whatever was there. The data is padded to `page_size` by `write_page`.
4. **Update metadata** — writes the same `root`, `height`, `total_keys`, and `next_free`, but replaces `free_head` with `page_num`. This makes the freed page the new head of the list.

The key insight: the free list is **intrusive** — each free page stores its own "next" pointer in its body, so no separate data structure is needed.

### Side Effects

- **Two disk writes**: one to overwrite the freed page's contents, one to update the metadata page.
- **Two `flush()` calls** (inside `write_page` and `write_meta`).
- Increments `self.pages_written` by 1 (the metadata write goes through `_write_meta`, which bypasses the counter; only the `write_page` call increments it).
- **No WAL logging**: this method operates at the `PageManager` level, below the WAL. The `BTree._delete` method that calls it wraps the surrounding operations in WAL entries, but the `free_page` call itself is not individually WAL-protected. If a crash occurs between the `write_page` and `write_meta` calls, the page is overwritten but the metadata still points to the old free head — the freed page becomes a leaked orphan.

### Error Handling

None. If the file I/O fails, the underlying `write`/`seek` calls raise `OSError` and the free list is left in a partially-updated state. There is no rollback or atomicity guarantee at this level.

### Usage Patterns

The only caller in this codebase is `BTree._delete` at line ~247, which frees an empty leaf after merging its sibling pointer:

```python
self.pm.free_page(child_page)
```

The caller must:
1. First unlink the page from the tree structure (update parent's children list, fix sibling pointers).
2. Then call `free_page`.
3. The corresponding `allocate_page` method checks `free_head != NO_SIBLING` before allocating from the high-water mark, so freed pages are reused LIFO (most-recently-freed page is allocated first).

### Dependencies

- `struct` — for packing the header and pointer in big-endian format.
- `self.read_meta()` / `self.write_meta()` — to atomically read and update the free-list head pointer.
- `self.write_page()` — to overwrite the freed page with the free-list node.
- Constants `HEADER_FMT` (3 bytes: `>BH`) and implicitly `META_FMT` (20 bytes: `>5I`).

### Assumptions Not Enforced

1. **No double-free detection** — freeing the same page twice creates a cycle in the free list, causing `allocate_page` to return the same page number repeatedly, silently corrupting the tree.
2. **No bounds checking** — `page_num` could be 0 (the metadata page) or beyond the file's extent; both would corrupt state.
3. **No crash atomicity** — the `write_page` and `write_meta` are not atomic together. A crash between them leaks the page.
4. **Free-list node is typed as `INTERNAL` with 0 keys** — `allocate_page` doesn't validate the page type when popping from the free list, so this works, but nothing prevents confusing a free-list node with a real empty internal node during debugging or recovery.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:allocate_page` — The allocation counterpart; shows how LIFO reuse works and how free-list nodes are popped
- [function] `b-tree-storage-engine/btree.py:_delete` — The only caller of `free_page`; shows how page unlinking and sibling-pointer fixup happen before freeing
- [function] `b-tree-storage-engine/btree.py:_wal_write_page` — The WAL-protected write path that `free_page` notably does *not* use, exposing a crash-safety gap
- [general] `intrusive-free-list-pattern` — The technique of storing allocator metadata inside the free blocks themselves, common in OS kernels and database page managers
- [file] `b-tree-storage-engine/test_btree.py` — Check whether delete-then-reinsert tests exercise the free-list reuse path

---

## Beliefs

- `free-list-is-lifo` — The page free list is LIFO: `free_page` pushes to the head, `allocate_page` pops from the head, so the most recently freed page is reused first.
- `free-page-not-wal-protected` — `PageManager.free_page` performs two non-atomic writes without WAL logging; a crash between them leaks the freed page permanently.
- `free-list-intrusive-storage` — Free-list "next" pointers are stored inside the free pages themselves at offset `HEADER_SIZE` (byte 3), requiring no separate allocation metadata structure.
- `no-double-free-detection` — Calling `free_page` twice on the same page creates a cycle in the free list, which `allocate_page` cannot detect, leading to silent corruption.
- `free-page-only-called-from-delete` — In this codebase, `free_page` is only invoked by `BTree._delete` when removing an emptied leaf node from the tree.

