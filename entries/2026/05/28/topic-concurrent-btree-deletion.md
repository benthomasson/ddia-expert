# Topic: Blink-tree latch coupling during delete/merge is where real databases spend most of their complexity; this single-threaded impl sidesteps it entirely

**Date:** 2026-05-28
**Time:** 19:19

I'll work from the observations provided. The grep results and partial file reads give a clear picture.

---

# Blink-tree Latch Coupling: Complexity This Implementation Avoids

## The core insight

This B-tree implementation has **no delete-merge logic, no latch coupling, and no concurrency control** — and this is the most important thing to understand about its design tradeoffs relative to real database B-trees.

### What's missing — and why it matters

Three grep searches tell the story:

| Pattern searched | Matches |
|---|---|
| `merge\|redistribute\|underflow\|rebalance` | **0** |
| `lock\|latch\|mutex\|thread\|concurrent` | **0** |
| `sibling\|right_ptr\|link\|blink\|next_leaf` | **0** |

Zero hits across all three. This implementation has no concept of node merging, no concurrency primitives, and no Blink-tree sibling pointers.

### What the delete actually does

The test file (`test_btree.py`, lines 126–148, `test_delete_and_reinsert`) confirms that delete works — it removes keys and adjusts the count. The test `test_delete_frees_empty_leaf` (line 177+) shows that completely emptied leaf pages get freed. But the grep for `merge|redistribute|underflow|rebalance` returning zero matches means: **when a leaf becomes half-empty after a delete, nothing happens.** The tree just leaves sparse nodes in place. Only when a leaf is fully emptied does it get reclaimed via `PageManager.free_page()` (line 92–97 in `btree.py`), which pushes it onto a free list.

### Why real databases can't do this

In a concurrent B-tree (Blink-tree variant), delete is the hardest operation because of **latch coupling** — the protocol where you hold a latch on a parent while acquiring a latch on a child, then release the parent. During a merge:

1. You latch the parent to read the child pointer
2. You latch the child and discover it's underflowed
3. You need to latch the **sibling** to merge with it
4. But the sibling might be a child of a **different parent** (after prior splits)
5. You may need to latch that other parent too — and now you risk deadlock
6. The parent's separator key must be updated or removed atomically with the merge
7. If the parent itself underflows, the merge cascades upward

This is where Blink-tree link pointers (the `NO_SIBLING = 0xFFFFFFFF` sentinel defined at line 12) were meant to help — sibling links let you traverse right without going back through the parent, breaking the deadlock cycle. But this implementation only defines the constant; the grep confirms `sibling` appears nowhere in the actual tree logic. The `_serialize_leaf` function (line 162) writes a `next_sibling` field into the page format, but it's structural scaffolding that's never used for concurrent traversal.

### The single-threaded escape hatch

Because this implementation is single-threaded:
- No latches needed — there's no concurrent reader that could see a half-merged node
- No latch coupling protocol — you just walk down the tree, do your work, walk back up
- No sibling-pointer traversal during merge — there is no merge
- No deadlock risk — one thread, no locks, no deadlock

The WAL (`btree.py`, lines 109–157) provides crash safety, and `PageManager` provides page-level I/O, but neither deals with concurrent access. `WAL.log_write` (line 127) does `os.fsync` for durability, not for inter-thread visibility.

### The cost of this simplification

The tradeoff is real: after many deletes, the tree accumulates sparse leaf pages that waste I/O. A range scan reads every leaf along the chain regardless of how few keys each one holds. A production B-tree would merge underflowed siblings to maintain a minimum fill factor (typically 50%), keeping range scan performance predictable. This implementation lets fill factor degrade unboundedly after deletes.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:free_page` — The only reclamation path after deletion; understanding the free list reveals what happens to completely emptied pages versus merely sparse ones
- [general] `blink-tree-paper` — Lehman and Yao's 1981 paper "Efficient Locking for Concurrent Operations on B-Trees" defines the sibling-link protocol this implementation alludes to but doesn't need
- [file] `b-tree-storage-engine/btree.py` — Read the full delete method (not captured in the 200-line window) to confirm it does simple key removal without underflow checks
- [general] `latch-coupling-postgres` — PostgreSQL's `nbtree` README documents their actual Blink-tree latch coupling protocol, showing the complexity this implementation avoids
- [function] `b-tree-storage-engine/btree.py:_serialize_leaf` — The `next_sibling` field is written but apparently unused; understanding whether range scans use it or walk through the parent clarifies the structural gap

## Beliefs

- `delete-has-no-merge` — Deleting a key removes it from its leaf but never triggers merging or redistribution with sibling nodes; underflowed pages persist until fully empty
- `no-concurrency-primitives` — The entire B-tree implementation contains zero locking, latching, mutex, or thread-safety constructs; it is strictly single-threaded
- `free-list-reclaims-empty-leaves` — `PageManager.free_page()` pushes emptied pages onto a singly-linked free list (next pointer stored in the first 4 bytes after header); `allocate_page()` pops from this list before extending the file
- `sibling-field-is-structural-only` — Leaf pages carry a `next_sibling` field in their serialized format (`NO_SIBLING = 0xFFFFFFFF` sentinel), but no tree operation uses sibling pointers for traversal or merging
- `wal-provides-crash-safety-not-concurrency` — The WAL syncs with `os.fsync` for durability guarantees; it has no role in coordinating concurrent access, consistent with the single-threaded design

