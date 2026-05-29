# Topic: Insert/delete cycles should show pages being reclaimed from the free list rather than the file growing unboundedly

**Date:** 2026-05-29
**Time:** 07:41

# Page Reclamation via the Free List in Insert/Delete Cycles

## The Core Mechanism

The B-tree maintains a **free list** — a singly-linked list of deallocated pages stored within the data file itself. This is what prevents unbounded file growth during insert/delete cycles. Two methods on `PageManager` implement this:

### Freeing a page (`btree.py:97-102`)

```python
def free_page(self, page_num):
    root, height, total_keys, next_free, free_head = self.read_meta()
    data = struct.pack(HEADER_FMT, 0, 0) + struct.pack('>I', free_head)
    self.write_page(page_num, data)
    self.write_meta(root, height, total_keys, next_free, page_num)
```

When a page is freed, its content is overwritten with a minimal header plus a pointer to the **previous** free list head. The metadata's `free_list_head` is then updated to point to this newly freed page. This is a classic **push onto a singly-linked stack** — the freed pages form a chain where each page's body contains the pointer to the next free page.

### Allocating a page (`btree.py:87-95`)

```python
def allocate_page(self):
    root, height, total_keys, next_free, free_head = self.read_meta()
    if free_head != NO_SIBLING:
        page_num = free_head
        page_data = self.read_page(page_num)
        new_head = struct.unpack('>I', page_data[HEADER_SIZE:HEADER_SIZE+4])[0]
        self.write_meta(root, height, total_keys, next_free, new_head)
        return page_num
    page_num = next_free
    self.write_meta(root, height, total_keys, page_num + 1, free_head)
    return page_num
```

Allocation checks `free_head` first. If it's not `NO_SIBLING` (`0xFFFFFFFF`), it **pops** the head page off the free list by reading the chained pointer from that page's body and updating the metadata. The file does not grow. Only when the free list is empty does it fall through to bumping `next_free` — the high-water mark that extends the file.

## The Metadata Layout

The metadata page (page 0, `btree.py:17-18`) stores five 4-byte unsigned integers:

| Field | Purpose |
|-------|---------|
| `root_page` | Page number of the B-tree root |
| `height` | Tree height |
| `total_keys` | Key count |
| `next_free_page` | High-water mark — next page number if file must grow |
| `free_list_head` | Head of free list, or `NO_SIBLING` if empty |

The distinction between `next_free_page` and `free_list_head` is critical. `next_free_page` only increases (it marks how far the file has ever extended). `free_list_head` is the recycling mechanism that prevents `next_free_page` from being the only allocation path.

## What the Test Verifies

The test `test_delete_frees_empty_leaf` (`test_btree.py:172`) sets up exactly this scenario: insert 10 keys to build a height-2 tree with multiple leaves, then delete keys from a leaf until it's empty. The test captures `pages_before = tree.stats.total_pages` just before the deletion cycle begins — it's clearly intended to verify that the emptied leaf page is returned to the free list rather than left as dead space.

## What's Missing from These Observations

The observations only include the first 200 lines of `btree.py` (612 total) and the first 200 of `test_btree.py` (300 total). The **delete method itself** — the logic that decides when a leaf is empty enough to free — lives in the unseen portion of `btree.py`. Specifically:

- The `BTree.delete()` method and its recursive `_delete()` helper
- Any merge or redistribution logic (sibling stealing, node merging) that triggers `free_page()`
- Whether the implementation does full B-tree merge/redistribute or simply frees completely empty leaves

The grep results all returned zero matches, which suggests the search was scoped incorrectly (possibly searching outside the `b-tree-storage-engine/` directory). The code clearly contains `free_list`, `free_page`, `allocate`, and `reuse` logic in the visible portion.

## The Insert/Delete Cycle

Putting it together, a well-functioning cycle looks like this:

1. **Insert phase**: Keys are added, leaves split, `allocate_page()` extends the file via `next_free_page` (no free pages yet).
2. **Delete phase**: Keys are removed, leaves empty out, `free_page()` pushes emptied pages onto the free list.
3. **Re-insert phase**: New keys are added, `allocate_page()` **pops from the free list** — `next_free_page` stays unchanged, the file does not grow.

If the file grew during re-insertion, it would mean `free_page()` was never called during deletion — a bug where the delete path removes keys from leaves but never reclaims the empty pages.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:delete` — The delete implementation and what triggers `free_page()` — this is the missing half of the reclamation story
- [function] `b-tree-storage-engine/btree.py:_serialize_leaf` — How leaf pages are packed and what determines when a leaf is "empty enough" to free
- [file] `b-tree-storage-engine/test_btree.py` — The remainder of `test_delete_frees_empty_leaf` (line 172+) and any other free-list tests that verify reclamation under cycles
- [general] `merge-vs-lazy-free` — Whether the implementation does full B-tree node merging (redistributing keys between siblings) or takes the simpler approach of only freeing completely empty leaves
- [general] `free-list-fragmentation` — How internal fragmentation behaves when freed pages are scattered across the file — the file never shrinks, only recycles interior pages

## Beliefs

- `free-list-is-intrusive-singly-linked` — The free list is stored within the freed pages themselves: each freed page's body contains a 4-byte pointer to the next free page, forming a singly-linked stack rooted at `free_list_head` in the metadata page.
- `allocate-prefers-free-list` — `PageManager.allocate_page()` always pops from the free list before extending the file via `next_free_page`, guaranteeing that reclaimed pages are reused before the file grows.
- `next-free-page-is-monotonic` — The `next_free_page` metadata field only ever increases; file size is never reduced, but interior pages are recycled through the free list.
- `free-page-overwrites-content` — `PageManager.free_page()` overwrites the freed page's content with a zero header and a free-list pointer, destroying the original leaf/internal data — freed pages cannot be accidentally read as valid tree nodes.

