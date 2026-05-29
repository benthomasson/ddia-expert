# Topic: PostgreSQL's two-phase page deletion (half-dead → dead) is the mechanism that makes deferred cleanup crash-safe; understanding it reveals why "just free the page" (as in `btree.py`) is insufficient for production

**Date:** 2026-05-29
**Time:** 07:48

# Two-Phase Page Deletion: Why "Just Free the Page" Breaks Under Crashes

## The Problem in `btree.py`

Look at `PageManager.free_page` (lines 89–94 of `btree.py`):

```python
def free_page(self, page_num):
    root, height, total_keys, next_free, free_head = self.read_meta()
    data = struct.pack(HEADER_FMT, 0, 0) + struct.pack('>I', free_head)
    self.write_page(page_num, data)
    self.write_meta(root, height, total_keys, next_free, page_num)
```

This does two things in sequence:
1. Overwrites the page with a free-list pointer
2. Updates the metadata page to make this page the new free-list head

If the process crashes between steps 1 and 2, the page's content is destroyed but it's not on the free list — it becomes a **leaked page**, permanently unreachable. Crash between 2 and the parent pointer update, and you have a **dangling reference**: the parent still points to a page that's now on the free list and could be reallocated to hold completely unrelated data.

The WAL (lines 112–153) doesn't help here either. It logs page writes and replays them on recovery, but the WAL has no concept of multi-step structural operations. It replays whatever was written — if only one of the two writes made it to the WAL before the crash, recovery faithfully reproduces a half-finished deletion.

## What PostgreSQL Does Instead

PostgreSQL's B-tree splits page deletion into two atomic phases with a recoverable intermediate state:

### Phase 1: Mark Half-Dead

The leaf page is marked **half-dead** in a single WAL-logged operation. The page's content is preserved. The parent's downlink to this page is removed atomically with the half-dead marking. Crucially, sibling links are **not** yet updated — other pages still point to this half-dead page.

This is safe because:
- A scan that follows a sibling link to a half-dead page can detect the flag and skip it
- The page's data hasn't been destroyed, so no information is lost
- If we crash here, recovery sees a half-dead page and knows to finish the job

### Phase 2: Unlink and Recycle (Dead)

A separate operation — often run by VACUUM, not the original deleter — updates the sibling links to skip the half-dead page, then moves the page to the free space map. Only after the sibling pointers are updated is the page truly **dead** and eligible for reuse.

If we crash between phases, the half-dead flag is a durable breadcrumb. On recovery, PostgreSQL finds half-dead pages and completes their deletion. No leaked pages, no dangling pointers.

## Why the Two-Phase Design Matters

The core insight is that page deletion requires **three distinct structural changes** that can't all fit in one atomic disk write:

| Step | What changes | `btree.py` | PostgreSQL |
|------|-------------|-----------|------------|
| Remove parent downlink | Parent page | Not done at all in `free_page` | Phase 1 (WAL-atomic with half-dead flag) |
| Update sibling links | Left and right neighbor pages | Not done (grep for `sibling` in deletion path returns nothing) | Phase 2 (separate WAL record) |
| Add to free list | Metadata/FSM | `free_page` line 93–94 | Phase 2 (after siblings updated) |

`btree.py` jumps straight to the third step — look at the grep results: there are **zero matches** for `merge`, `rebalance`, `underflow`, `redistribute`, or `sibling` in any deletion context. The delete operation (referenced in `test_delete_and_reinsert`, line 155) removes the key from the leaf but never restructures the tree. Empty leaves persist as wasted space, or are freed without fixing the pointers that reference them.

The grep for `lock|latch|concurrent|thread` also returns nothing. In PostgreSQL, the two-phase design exists partly because concurrent readers might be traversing sibling links during deletion. But even in a single-threaded system, the crash-safety argument holds: you need a recoverable intermediate state between "page is part of the tree" and "page is on the free list."

## The Deeper Lesson

The two-phase protocol is an instance of a general pattern: **any multi-page structural change needs a crash-recoverable intermediate state**. PostgreSQL's half-dead flag is essentially a micro-transaction marker embedded in the page itself. The WAL ensures the flag is durable; the flag tells recovery what operation was in progress; and the second phase is idempotent so it can be safely retried.

`btree.py`'s WAL (lines 112–153) logs individual page writes but has no concept of operation boundaries. There's no `begin_structural_change` / `end_structural_change` bracketing. The `commit` method (line 136) just truncates the entire WAL — it's all-or-nothing for the whole transaction, not per-operation. This works for simple puts (write leaf + write metadata), but falls apart for multi-step tree restructuring where you need the WAL to record *intent*, not just *data*.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:free_page` — Trace every caller to understand which deletion scenarios actually reclaim pages and which leak them
- [general] `postgres-nbtree-vacuum` — PostgreSQL's `src/backend/access/nbtree/nbtpage.c` contains `_bt_pagedel` and `_bt_unlink_halfdead_page`, the canonical implementation of this two-phase protocol
- [general] `wal-operation-boundaries` — How production systems (PostgreSQL, SQLite, InnoDB) encode multi-step structural operations in their WAL so recovery can complete or roll back partial work
- [function] `b-tree-storage-engine/btree.py:allocate_page` — The free-list reuse path (lines 82–88) trusts that freed pages are safe to reuse; understanding how a corrupted free list leads to data loss completes the picture
- [general] `sibling-pointer-maintenance` — Why B-link trees maintain sibling pointers and how their invariants interact with concurrent access and crash recovery

## Beliefs

- `btree-free-page-no-parent-fixup` — `PageManager.free_page` adds a page to the free list without removing the parent's downlink to it, creating a dangling-pointer risk on crash
- `btree-no-merge-or-rebalance` — The implementation has no merge, rebalance, or redistribution logic; deletes remove keys from leaves but never restructure the tree
- `btree-wal-no-operation-boundaries` — The WAL logs individual page writes and commits by full truncation; it cannot represent a multi-step structural operation as an atomic unit
- `btree-single-threaded-assumption` — The implementation has no locking, latching, or concurrency control; all crash-safety gaps exist even without concurrent access

