# Topic: PostgreSQL's `src/backend/access/nbtree/nbtpage.c` contains `_bt_pagedel` and `_bt_unlink_halfdead_page`, the canonical implementation of this two-phase protocol

**Date:** 2026-05-29
**Time:** 10:46

# Two-Phase B-Tree Page Deletion: What PostgreSQL Does vs. What This Codebase Does

The claim references PostgreSQL's canonical approach to safely removing pages from a B-tree: first mark a page as "half-dead" (severing it from its parent but keeping sibling links intact), then unlink it from the sibling chain in a second step. This two-phase protocol exists because a single atomic operation can't safely update the parent, left sibling, and right sibling simultaneously — a crash mid-operation could leave the tree with dangling pointers.

**This codebase does not implement any form of two-phase page deletion.** The observations confirm this comprehensively:

- Grepping for `half.?dead`, `unlink`, `merge`, `rebalance`, `underflow`, and `redistribute` yields **zero matches** in the B-tree module.
- The `_delete` method (`btree.py`, roughly lines 442–458 per `fix-plan.md`) removes keys from leaf nodes but **never frees empty pages and never touches parent or sibling pointers**.
- `fix-plan.md` (lines 51–60) documents this explicitly as **Bug 5**: _"Delete doesn't free pages… The free list mechanism exists but is unused by delete."_
- The proposed fix in the plan is deliberately minimal — free empty leaves and remove the parent pointer, but explicitly states: _"Full merge/redistribute is out of scope."_

## What the codebase does have

The infrastructure for page freeing exists but is disconnected from delete:

- **`PageManager.free_page`** (lines ~93–98) pushes a page onto a free list by writing a pointer to the old head and updating metadata.
- **`PageManager.allocate_page`** (lines ~85–92) pops from the free list when reusing pages, falling back to appending new pages.
- **Leaf sibling pointers** are maintained (`next_sibling` in `_serialize_leaf`, line ~188; `NO_SIBLING = 0xFFFFFFFF` sentinel, line 13), but no code updates the *previous* sibling's forward pointer when a leaf is removed.
- **No locking or latching** — grep for `lock`, `latch`, `concurrent`, `atomic` returns zero matches. The two-phase protocol exists in PostgreSQL largely *because* of concurrent access; without concurrency, the simpler approach is to just do the unlink atomically with the WAL.

## Why two phases matter (and why this implementation can skip it — for now)

PostgreSQL's `_bt_pagedel` marks a page half-dead in one WAL-logged operation, then `_bt_unlink_halfdead_page` removes it from the sibling chain in a separate WAL-logged operation. If a crash occurs between phases, recovery can see the half-dead marker and complete the unlink. This is necessary because PostgreSQL has concurrent readers traversing sibling chains — a reader following a `next` pointer must never land on a freed page.

This implementation is single-threaded with no concurrent readers, so the two-phase protocol isn't strictly necessary. But the absence of *any* page freeing on delete means the tree only grows — a significant practical limitation documented in Bug 5.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_delete` — The current delete implementation that only removes keys without freeing pages or updating the tree structure
- [function] `b-tree-storage-engine/btree.py:free_page` — The free-list mechanism that exists but is never called from delete, showing the gap between infrastructure and usage
- [file] `b-tree-storage-engine/fix-plan.md` — Bug 5 describes the planned minimal fix (free empty leaves, update parent) and explicitly scopes out merge/redistribute
- [general] `btree-concurrent-delete-safety` — How concurrent readers traversing sibling chains interact with page deletion, the core reason PostgreSQL needs two phases
- [general] `wal-atomicity-for-multi-page-updates` — How WAL logging could make a single-phase unlink safe in this codebase since there are no concurrent readers

## Beliefs

- `btree-delete-never-frees-pages` — `_delete` removes keys from leaves but never calls `free_page`, so the B-tree file only grows; the free list infrastructure in `PageManager` is entirely unused by the delete path
- `btree-no-concurrency-control` — The B-tree implementation has no locks, latches, or atomic operations, making the two-phase half-dead protocol unnecessary but also making the tree unsafe for concurrent access
- `btree-free-list-exists-unused` — `PageManager.free_page` and `allocate_page` implement a working free-list via a singly-linked chain in page headers, but nothing in the tree operations ever calls `free_page`
- `btree-leaf-sibling-chain-forward-only` — Leaf pages store a `next_sibling` forward pointer (`NO_SIBLING` = `0xFFFFFFFF` sentinel) but no backward pointer, so unlinking a leaf from the chain requires finding the predecessor by traversal from the parent

