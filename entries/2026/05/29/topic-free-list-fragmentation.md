# Topic: How internal fragmentation behaves when freed pages are scattered across the file — the file never shrinks, only recycles interior pages

**Date:** 2026-05-29
**Time:** 10:30

# Internal Fragmentation in the B-Tree Storage Engine

## How the Free List Works

The B-tree uses a **fixed-size page** model (`PageManager`, `btree.py:28`) where every page — whether it holds a leaf, an internal node, or nothing at all — occupies exactly `page_size` bytes (default 4096) on disk. The file is a flat array of these fixed slots.

When a page is no longer needed (e.g., a leaf is emptied by deletes), it's returned to a **singly-linked free list** embedded in the file itself:

**Freeing a page** (`free_page`, `btree.py:96–101`):
```python
def free_page(self, page_num):
    root, height, total_keys, next_free, free_head = self.read_meta()
    data = struct.pack(HEADER_FMT, 0, 0) + struct.pack('>I', free_head)
    self.write_page(page_num, data)
    self.write_meta(root, height, total_keys, next_free, page_num)
```

The freed page's content is overwritten with a small header (3 bytes) plus a 4-byte pointer to the previous free list head. The remaining ~4089 bytes of that page are **dead space** — zeroed out by the `ljust` padding in `write_page` (`btree.py:81`). The metadata page's `free_head` field is updated to point to this newly freed page, forming a LIFO (stack) linked list.

**Allocating a page** (`allocate_page`, `btree.py:86–95`):
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

Free list pages are preferred over extending the file. Only when `free_head == NO_SIBLING` (the sentinel `0xFFFFFFFF`, line 13) does allocation bump `next_free` and grow the file.

## Why the File Never Shrinks

There is **no `truncate`, `compact`, or `defrag` operation** anywhere in the codebase. The grep results for those patterns return zero matches. Once the file grows to N pages, it stays at N pages forever. Deleting half your data doesn't reclaim a single byte of disk space — it just adds pages to the free list.

## The Fragmentation Pattern

Consider what happens after a workload like insert 1000 keys, then delete every other key:

1. **The file** contains, say, 300 pages of actual data interleaved with 150 freed pages scattered throughout.
2. **The free list** is a linked list threading through those 150 scattered positions — page 247 → page 132 → page 58 → ... — in LIFO order of when they were freed, **not** in file-offset order.
3. **New allocations** reuse those scattered pages, so a subsequent range scan may need to seek to page 3, then page 247, then page 58 — destroying any sequential I/O pattern.

This is **internal fragmentation** in the classic sense: the file has allocated space that cannot be used by the OS for anything else, and the live data is scattered across it in a pattern that defeats readahead and sequential prefetch.

### The LIFO Problem Amplifies Scatter

Because `free_page` pushes onto the head and `allocate_page` pops from the head (stack discipline), pages freed in a batch are reused in reverse order. If you delete keys `k00` through `k09` (left-to-right in the tree), then insert `k20` through `k29`, the new keys land in the pages that held `k09`, `k08`, `k07`... — the reverse order. Over many delete-reinsert cycles, the logical key order and physical page order become completely decorrelated.

### Quantifying the Waste

Each free-list page wastes nearly the entire page — only 7 bytes are meaningful (3-byte header + 4-byte next pointer), while the remaining `page_size - 7` bytes (4089 bytes at default settings) are padding. If you have 150 free pages at 4KB each, that's ~600KB of disk holding nothing but linked-list pointers.

The test `test_delete_frees_empty_leaf` (`test_btree.py:168`) captures the `pages_before` count and verifies pages move to the free list, but notably **does not assert** that the file size decreased — because it can't. The test validates recycling, not reclamation.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:allocate_page` — Trace what happens when the free list is exhausted vs. when it has entries, and how this affects file growth patterns
- [function] `b-tree-storage-engine/btree.py:free_page` — Study how the LIFO ordering of the free list affects physical locality of reused pages
- [general] `free-list-ordering-strategies` — Compare LIFO free lists (used here) against sorted/buddy-system allocators that preserve spatial locality
- [general] `online-compaction` — How production B-trees (SQLite, PostgreSQL, InnoDB) solve this with VACUUM, autovacuum, or page-level compaction without taking the database offline
- [file] `b-tree-storage-engine/test_btree.py` — The test suite validates recycling but never asserts file-size reduction — a good place to add a fragmentation-ratio test

## Beliefs

- `free-list-is-lifo-stack` — The free page list is a LIFO stack embedded in the data file itself; `free_page` pushes to head and `allocate_page` pops from head, so reuse order is reverse of free order
- `file-never-shrinks` — The data file only grows (via `next_free` bump in `allocate_page`) and never truncates or compacts; no shrink/defrag/compact code exists
- `free-page-wastes-full-page` — Each free-list entry occupies an entire page (default 4096 bytes) but stores only 7 bytes of useful data (3-byte header + 4-byte next pointer)
- `free-list-preferred-over-growth` — `allocate_page` always checks `free_head != NO_SIBLING` before bumping `next_free`, so the file only grows when the free list is empty

