# Topic: The technique of storing allocator metadata inside the free blocks themselves, common in OS kernels and database page managers

**Date:** 2026-05-29
**Time:** 10:50

# Storing Allocator Metadata Inside Free Blocks

## The Core Idea

When a page allocator needs to track which pages are free, it has two choices: maintain a separate data structure (an array, bitmap, or external list), or store the bookkeeping *inside the free pages themselves*. This codebase uses the second approach — an **intrusive free list** — where each freed page's own bytes hold the pointer to the next free page.

This works because a free page isn't being used for anything. Its 4096 bytes are wasted space. So the allocator repurposes a few of those bytes to thread a singly-linked list through the free pages.

## How It Works in This Codebase

The implementation lives in `PageManager` in `b-tree-storage-engine/btree.py`.

### The metadata page (page 0) holds the list head

Line 16 defines the metadata layout:

```
#   root_page(4B), height(4B), total_keys(4B), next_free_page(4B), free_list_head(4B)
```

The fifth field, `free_list_head`, is the page number of the first free page. It's the entry point into the linked list. If no pages are free, it holds the sentinel `NO_SIBLING` (`0xFFFFFFFF`, line 12).

### Freeing a page (line 96–101): push onto the list

```python
def free_page(self, page_num):
    root, height, total_keys, next_free, free_head = self.read_meta()
    # Write a free-list node: header + pointer to old head
    data = struct.pack(HEADER_FMT, 0, 0) + struct.pack('>I', free_head)
    self.write_page(page_num, data)
    self.write_meta(root, height, total_keys, next_free, page_num)
```

This is a classic stack push. It:
1. Reads the current head of the free list (`free_head`)
2. **Overwrites the freed page's contents** with a minimal header (type=0, num_keys=0) followed by a 4-byte pointer to the old head — this is the metadata stored inside the free block
3. Updates the metadata page so `free_list_head` now points to the newly freed page

After this, the freed page's bytes look like:

```
[1B type=0] [2B num_keys=0] [4B next_free_pointer] [... rest is padding ...]
```

The key insight: the `next_free_pointer` lives at bytes 3–6 of the freed page itself. No external table needed.

### Allocating a page (line 82–93): pop from the list

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

This is a stack pop. It:
1. Checks if the free list is non-empty (`free_head != NO_SIBLING`)
2. **Reads the free page to extract the next pointer** from `page_data[HEADER_SIZE:HEADER_SIZE+4]` — pulling the allocator metadata back out of the block
3. Updates the metadata page so `free_list_head` advances to the next free page
4. Returns the reclaimed page number

If the list is empty, it falls back to bumping `next_free` — extending the file.

## Why This Design

**Zero overhead for free tracking.** The free list consumes no additional pages or memory. Each free page is its own list node. If you have 1000 free pages, you have a 1000-node linked list that occupies exactly those 1000 pages — no extra storage.

**O(1) allocate and free.** Both operations are stack push/pop: read the head, follow one pointer, update the head. Compare this to scanning a bitmap, which is O(n) in the number of pages.

**The tradeoff is fragmentation.** A free list has no notion of contiguity. If you free pages 5, 10, and 15, subsequent allocations return them in LIFO order (15, 10, 5) regardless of physical layout. Bitmap-based allocators can search for contiguous runs, which matters for sequential I/O. This implementation accepts that tradeoff — B-tree access patterns are already random I/O by nature.

## Where This Pattern Appears More Broadly

This is the same technique used by `malloc` implementations (glibc's ptmalloc stores forward/backward pointers in freed chunks), Linux kernel page allocators (buddy system free lists), and production databases like PostgreSQL (which threads free pages into a free space map). The principle is universal: if a block isn't holding user data, its bytes are available for the allocator's own bookkeeping.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:allocate_page` — Trace the two allocation paths (free list reuse vs. file extension) and when each triggers
- [general] `free-list-fragmentation` — How LIFO reuse ordering affects page locality and what strategies (e.g., address-ordered free lists, buddy systems) mitigate it
- [general] `crash-safety-of-free-list` — The current `free_page` writes the page then updates metadata in two non-atomic steps — explore what happens if a crash occurs between them
- [function] `b-tree-storage-engine/btree.py:free_page` — Called from `_delete_entry` (line 472) during B-tree key deletion; follow the deletion path to see when pages become reclaimable
- [general] `bitmap-vs-freelist-allocators` — Compare this linked-list approach to bitmap-based allocators (e.g., ext4 block groups) for understanding the contiguity vs. simplicity tradeoff

## Beliefs

- `free-list-is-intrusive-singly-linked` — The free page list is a singly-linked list where each free page stores the next-free pointer at bytes `HEADER_SIZE` to `HEADER_SIZE+4` within the page itself
- `allocate-page-prefers-reuse` — `allocate_page` checks the free list before extending the file, so freed pages are always reused before new pages are appended
- `free-list-is-lifo` — `free_page` pushes onto the head and `allocate_page` pops from the head, making reuse order LIFO (most recently freed page is allocated first)
- `free-list-head-persisted-in-page-zero` — The free list head pointer is stored as the fifth field of the metadata page (page 0), making it durable across restarts
- `free-page-not-atomic-with-meta-update` — `free_page` performs two separate writes (page content then metadata) without WAL protection, creating a window where a crash could leak a page

