# Topic: Whether the implementation does full B-tree node merging (redistributing keys between siblings) or takes the simpler approach of only freeing completely empty leaves

**Date:** 2026-05-29
**Time:** 10:29

## B-tree Delete and Merge Strategy: Neither — Deletion Isn't Implemented

The short answer: this B-tree implementation **does not support deletion at all**. It implements neither full node merging nor the simpler empty-leaf-freeing approach, because there is no `delete` method anywhere in the codebase.

### Evidence

The grep results are definitive:

- **No delete method exists** — searching for `def delete|def _delete|def remove` across `b-tree-storage-engine/btree.py` returned zero matches.
- **No merge vocabulary exists** — searching for `merge|redistribute|rebalance|underflow|borrow|rotate` returned zero matches. None of the concepts needed for B-tree node merging appear anywhere in the code.
- **No delete tests exist** — searching for `def test.*delete|def test.*remove|def test.*merge` in the test files also returned zero matches.

### What the Infrastructure *Could* Support

The `PageManager` class does have the plumbing for page recycling (`btree.py`, lines ~89–97):

- `allocate_page()` checks a free list before extending the file
- `free_page(page_num)` links a page into a singly-linked free list stored in the page headers

And leaf nodes track a `next_sibling` pointer (`NO_SIBLING = 0xFFFFFFFF`, line 13), which is used for sequential leaf scanning — but this is for range queries, not for locating merge candidates.

So the page-level machinery for freeing nodes exists, but nothing in the B-tree logic calls `free_page()` as part of a delete-then-merge workflow. The free list is only used during splits (when a page allocated from the free list might later be freed if a WAL recovery rolls something back).

### What This Means

This is an **insert-and-lookup-only B-tree** — an append-only design where the tree grows but never shrinks. This is a common simplification in educational implementations. DDIA itself notes that many real-world storage engines (like LSM-tree-based systems) avoid in-place deletion entirely, using tombstones instead. This implementation skips even that — deleted keys simply can't be expressed.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:allocate_page` — Understand how the free list works for page reuse during splits, since this is the closest the code gets to page lifecycle management
- [function] `b-tree-storage-engine/btree.py:_serialize_leaf` — The `next_sibling` pointer in leaf serialization enables range scans but not merging — worth understanding the distinction
- [general] `btree-delete-underflow-handling` — Study how production B-trees (SQLite, PostgreSQL) handle underflow: borrowing from siblings vs. merging, and the cascade implications for parent nodes
- [file] `b-tree-storage-engine/btree.py` — Read the full file (lines 200–612) to understand what operations *are* implemented (insert, split, search, range scan) and confirm there's no hidden delete path
- [general] `tombstone-vs-physical-delete` — Compare this gap against LSM-tree deletion (tombstone markers) to understand why many storage engines avoid physical node removal entirely

## Beliefs

- `btree-no-delete-operation` — The B-tree implementation has no delete, remove, or key-removal method; it is insert-and-lookup only
- `btree-no-merge-or-redistribute` — No node merging, key redistribution, sibling borrowing, or underflow handling exists anywhere in the B-tree code
- `btree-free-list-unused-by-logic` — `PageManager.free_page()` provides page recycling infrastructure but no B-tree operation calls it as part of normal key management
- `btree-next-sibling-for-range-scans` — The `next_sibling` pointer in leaf nodes serves sequential range scanning, not sibling-aware merge operations

