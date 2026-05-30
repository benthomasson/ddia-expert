# Topic: This implementation only removes empty leaves at depth 2 and does not rebalance or merge underfull nodes, unlike textbook B-tree deletion — worth comparing with Knuth's algorithm or CLRS

**Date:** 2026-05-29
**Time:** 10:26

I can't access the target repo directly, but the observations provide enough to explain the core issue. The critical evidence is the grep results and the test names.

---

## Simplified B-Tree Deletion: Empty-Leaf Cleanup Only

### What the implementation does

The B-tree in `b-tree-storage-engine/btree.py` takes a shortcut on deletion: it removes the key from the leaf node, and if that leaf becomes completely empty (zero keys) and it's not the root, it frees the page. That's it.

The evidence is clear from two directions:

1. **The grep for `(merge|rebalance|underfull|underflow|redistribute|borrow|rotate)` returned zero matches** across the entire file (612 lines). None of these concepts exist in the codebase.

2. **The test `test_delete_frees_empty_leaf`** (line ~170 of `test_btree.py`) is named precisely: *"Deleting all keys from a non-root leaf should free the page."* This confirms the only structural cleanup is freeing completely emptied leaf pages.

The `PageManager.free_page` method (`btree.py:96-101`) implements this by linking freed pages into a singly-linked free list:

```python
def free_page(self, page_num):
    root, height, total_keys, next_free, free_head = self.read_meta()
    data = struct.pack(HEADER_FMT, 0, 0) + struct.pack('>I', free_head)
    self.write_page(page_num, data)
    self.write_meta(root, height, total_keys, next_free, page_num)
```

The freed page's body stores a pointer to the previous free list head, and the metadata page is updated. The corresponding `allocate_page` (lines ~84-93) reclaims from this list before extending the file.

### What textbook B-tree deletion does differently

In Knuth (TAOCP Vol. 3, §6.2.4) and CLRS (Chapter 18), deletion maintains a **minimum fill invariant**: every non-root node must have at least ⌈m/2⌉ keys (for order m). When a deletion causes a node to drop below this threshold, the algorithm takes corrective action:

| Situation | Textbook action | This implementation |
|-----------|----------------|---------------------|
| Leaf has ≥1 key after delete | No action needed | Same — no action |
| Leaf drops below min fill | **Borrow** from sibling or **merge** with sibling | Nothing — leaf stays underfull |
| Merge propagates underflow upward | **Recursive merge/borrow** up the tree | Nothing — internal nodes untouched |
| Root becomes empty after merge | **Shrink tree height** | Only frees empty leaves, never shrinks height |
| Internal node loses a child | **Rebalance** or **merge** | Never happens — internal nodes aren't modified on delete |

### Why this matters

The practical consequence is **space amplification under delete-heavy workloads**. If you insert 1000 keys into leaves of capacity 4, you get ~250 leaves. Delete 750 keys spread evenly, and you still have ~250 leaves, most holding only 1 key each. Textbook deletion would merge these down to ~63 leaves.

The tree also never shrinks in height. A tree that grew to height 4 during a burst stays at height 4 forever, even if 99% of keys are deleted. Every lookup still traverses 4 levels.

### Why the shortcut is reasonable here

This is a teaching implementation. The `max_keys_per_page=4` default in tests (line 8 of `test_btree.py`) makes this a pedagogical B-tree, not a production one. The implementation correctly handles the hard parts of B-tree storage — page-level I/O, splits, WAL crash recovery, sibling pointers for range scans — and punts on merge/rebalance, which is the most complex part of B-tree deletion (Knuth calls it "the most intricate" operation). That tradeoff makes sense for a DDIA reference: it demonstrates the storage engine concepts without the deletion complexity.

### What's missing from these observations

The actual `delete` method body (beyond line 200 of `btree.py`) was not captured. The file is 612 lines, and we only see the first 200. To fully confirm the "depth 2 only" claim from the original question, we'd need to read the delete logic and see whether it checks `height == 2` or has a more general empty-leaf check. The title of `test_delete_frees_empty_leaf` says "non-root leaf" generically, but the test body (also cut off) sets up a height-2 tree specifically.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:delete` — The actual delete implementation beyond line 200; confirms whether depth-2 is hardcoded or whether empty-leaf freeing works at any depth
- [function] `b-tree-storage-engine/btree.py:put` — The split logic is the counterpart to merge; understanding how splits propagate upward clarifies what merge *would* need to do in reverse
- [general] `btree-merge-algorithm` — CLRS Chapter 18.3 "Deleting a key from a B-tree" walks through the merge/borrow cases; compare the six cases there against what this implementation skips
- [file] `b-tree-storage-engine/test_btree.py` — The full `test_delete_frees_empty_leaf` body (beyond line 200) likely reveals the exact conditions under which page freeing is triggered
- [general] `space-amplification-after-deletes` — Write a test that inserts N keys, deletes N/2 spread evenly, then counts total allocated pages vs. minimum needed; quantifies the cost of skipping rebalance

---

## Beliefs

- `btree-no-merge-or-rebalance` — The B-tree implementation contains no merge, rebalance, borrow, or redistribute logic; the only structural change on deletion is freeing completely emptied leaf pages
- `btree-free-list-for-page-reuse` — Freed pages are linked into a singly-linked free list via the metadata page's `free_list_head`; `allocate_page` reclaims from this list before extending the file
- `btree-height-never-shrinks` — Tree height increases on root splits but never decreases on deletion, since there is no merge propagation that would collapse the root
- `btree-underfull-leaves-persist` — Leaf nodes with fewer keys than the minimum fill threshold remain in the tree indefinitely after deletions, causing space amplification proportional to the delete ratio

