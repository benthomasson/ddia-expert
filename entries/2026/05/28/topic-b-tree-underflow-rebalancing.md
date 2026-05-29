# Topic: Real B-trees (B+ trees specifically) merge or redistribute siblings on delete; this implementation skips that — explore what production databases do differently

**Date:** 2026-05-28
**Time:** 18:30

# B+ Tree Deletion: This Implementation vs. Production Databases

## What This Implementation Does

The delete path in `btree.py` (lines 442–458, referenced in `fix-plan.md` Bug 5) does the bare minimum: it finds the leaf containing the key, removes the key-value pair from that leaf, and returns. That's it.

The grep for `merge|redistribute|rebalance|underflow|borrow|sibling` across the codebase returns **zero matches**. There is no concept of a minimum occupancy invariant. A leaf can hold 1 key or 0 keys and nobody notices. Even freeing empty leaf pages was missing until Bug 5 in `fix-plan.md` identified it — and the planned fix explicitly states:

> Full merge/redistribute is out of scope — just handle the empty-leaf case.

So the fix plan itself acknowledges the gap and draws a boundary around it. After the fix, empty leaves get freed and their parent pointers cleaned up, but a leaf holding a single key in a node that could fit 4 (`max_keys_per_page=4`) will sit there half-empty forever.

## What Production B+ Trees Actually Do

Real B+ trees enforce a **minimum fill factor**: every non-root node must be at least half full. For a node with order *m*, that means at least ⌈m/2⌉ keys. When a deletion drops a node below this threshold, the tree must restore the invariant through one of two operations:

### 1. Redistribution (Borrowing from a Sibling)

If an adjacent sibling has more than the minimum number of keys, the underflowing node **borrows** one:

- For leaf nodes: move the leftmost (or rightmost) key-value pair from the sibling into the underflowing node, then update the separator key in the parent.
- For internal nodes: the separator key in the parent rotates down into the underflowing node, and a key from the sibling rotates up to replace it.

This is the preferred operation because it's **local** — it touches exactly three nodes (the underflowing node, one sibling, and the parent) and doesn't change the tree's height.

### 2. Merging (Coalescing with a Sibling)

If no sibling can spare a key (both are at minimum occupancy), the underflowing node **merges** with a sibling:

- Combine both nodes into one, pulling the separator key down from the parent (for internal nodes).
- Delete the now-redundant separator and child pointer from the parent.
- Free the empty page.

This is where it gets interesting: removing a key from the parent may itself cause an underflow, triggering a **cascade** of merges up the tree. If the root is left with a single child pointer and no keys, the child becomes the new root and the tree shrinks in height.

### How PostgreSQL Does It

PostgreSQL's B-tree implementation (`nbtree`) takes a pragmatic middle ground. It does **not** redistribute or merge on every delete. Instead:

- **Deleted keys are marked as dead** (index tuple flagged), not immediately removed.
- **VACUUM** later reclaims space, and only if a page becomes completely empty does it get removed from the tree via a "half-dead" → "dead" page state machine.
- Pages are never merged with siblings. Instead, empty pages are recycled into a free space map.
- This means PostgreSQL B-trees can have **sparse leaves** after heavy deletion — the fill factor degrades over time until VACUUM or `REINDEX` cleans up.

### How InnoDB (MySQL) Does It

InnoDB is closer to the textbook. Its B+ tree pages maintain a minimum fill factor, and deletions that drop below it trigger a merge with a sibling. InnoDB also uses an **insert buffer** (change buffer) to batch modifications to secondary indexes, amortizing the cost of rebalancing.

### How SQLite Does It

SQLite's B-tree (`btree.c`) performs full merge/redistribute on delete. When a cell is deleted and the page drops below roughly 1/3 full, it attempts to redistribute with siblings. If that fails, it merges. SQLite's approach is close to the textbook algorithm and benefits from its single-writer design — there's no concurrency complexity during rebalancing.

## Why This Matters

The minimum fill guarantee isn't academic pedantry. It provides:

| Property | With rebalancing | Without (this impl) |
|----------|-----------------|---------------------|
| **Space utilization** | Guaranteed ≥50% full | Can degrade to near-empty leaves |
| **Worst-case lookup** | O(log n) with tight constant | O(log n) but more pages to traverse |
| **Range scan perf** | Dense leaves, fewer page reads | Sparse leaves, more I/O for same key count |
| **Tree height** | Shrinks when appropriate | Never shrinks (root never demoted) |

The `test_delete_and_reinsert` test in `test_btree.py` (lines 142–164) exercises delete-then-reinsert but doesn't check that space was reclaimed or that the tree structure is optimal. The `test_delete_frees_empty_leaf` test (starting around line 177) only verifies that *completely* empty leaves get freed — not that half-empty ones get merged.

For this implementation — a teaching tool for DDIA concepts — skipping rebalancing is a reasonable trade-off. The tree stays correct (lookups still work, range scans via `next_sibling` linked-list pointers still work). It just wastes space and does slightly more I/O than necessary. The `NO_SIBLING = 0xFFFFFFFF` sentinel and the sibling-pointer infrastructure in `_serialize_leaf` (lines ~183+) are already there, which is exactly what you'd need for merge/redistribute — the plumbing exists, the logic doesn't.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_serialize_leaf` — The sibling pointer machinery is already serialized here; understanding this is prerequisite to implementing merge/redistribute
- [general] `postgres-nbtree-lazy-deletion` — PostgreSQL's approach of deferring rebalancing to VACUUM is an important production pattern that trades immediate consistency for throughput
- [file] `b-tree-storage-engine/fix-plan.md` — Bug 5's planned fix for empty-leaf freeing shows the parent-context-threading needed for any rebalancing, which is the hard part
- [general] `concurrent-btree-deletion` — Blink-tree latch coupling during delete/merge is where real databases spend most of their complexity; this single-threaded impl sidesteps it entirely
- [function] `b-tree-storage-engine/btree.py:allocate_page` — The free list reuse mechanism (lines ~86–95) that would receive pages from merge operations

## Beliefs

- `delete-never-rebalances` — The `_delete` method removes keys from leaves but never merges underfull nodes with siblings or redistributes keys, violating the standard B+ tree minimum-occupancy invariant
- `sibling-pointers-serialized-but-unused-for-merge` — Leaf pages serialize a `next_sibling` pointer (via `_serialize_leaf` and `NO_SIBLING` sentinel), providing the infrastructure for merge/redistribute, but no code path reads siblings during deletion
- `free-list-exists-but-delete-ignores-it` — `PageManager` implements `allocate_page`/`free_page` with a linked free list, but the delete path never calls `free_page` even when a leaf becomes empty (Bug 5 in fix-plan.md)
- `tree-height-never-shrinks` — Since delete never merges internal nodes or demotes the root, the tree height only increases (on splits) and never decreases, even if most keys are deleted

