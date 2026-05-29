# Topic: PostgreSQL's approach of deferring rebalancing to VACUUM is an important production pattern that trades immediate consistency for throughput

**Date:** 2026-05-28
**Time:** 19:17

# PostgreSQL's Deferred Rebalancing via VACUUM: A Production Pattern

## The Core Trade-off

When a B-tree delete drops a node below its minimum fill factor, the textbook answer is: rebalance immediately — merge with a sibling or borrow a key from one. This keeps the tree compact, guarantees at least 50% utilization, and preserves O(log n) lookups with a tight constant.

PostgreSQL's `nbtree` rejects this. Instead of rebalancing on every delete, it **marks deleted index tuples as dead**, leaves them in place, and lets **VACUUM** reclaim space later. Only when an entire page becomes empty does the page get removed — through a careful "half-dead" → "dead" state machine — and even then, the page is recycled into a free space map rather than merged with its sibling. Pages are never merged with siblings at all.

This is the defining production insight: **in a concurrent, high-throughput system, the cost of immediate rebalancing under contention far exceeds the cost of temporarily wasted space.**

## Why Immediate Rebalancing Is Expensive in Production

The entry on underflow rebalancing (`entries/2026/05/28/topic-b-tree-underflow-rebalancing.md:19-39`) lays out what the textbook algorithm requires:

1. **Redistribution** touches three nodes (underflowing node, sibling, parent) and requires updating the separator key in the parent.
2. **Merging** combines two nodes, pulls the separator down from the parent, frees a page, and can **cascade upward** — removing a key from the parent may cause *it* to underflow, triggering merges all the way to the root.

In a single-threaded implementation like the one in `btree.py`, this is just complexity. In a concurrent database, it's a latching nightmare. Every merge/redistribute requires exclusive locks on at least three pages simultaneously. Under concurrent read-write traffic, this creates contention hot spots on internal nodes — exactly the nodes most likely to be in every query's path. The entry at line 76 flags this explicitly:

> Blink-tree latch coupling during delete/merge is where real databases spend most of their complexity; this single-threaded impl sidesteps it entirely.

PostgreSQL's approach eliminates this problem: the delete path only needs to flag a tuple, not restructure the tree. VACUUM runs later — potentially during low-traffic periods — and handles cleanup without blocking readers (thanks to MVCC).

## How the Reference Implementation Compares

The `_delete` method in `btree.py` (documented in `entries/2026/05/28/b-tree-storage-engine-btree-_delete.md`) takes a third path: **no rebalancing, no deferred cleanup, just minimal immediate cleanup.** Specifically:

- It removes the key from the leaf immediately (lines 442–458)
- It uses a **three-valued return** (`False` / `True` / `'empty'`) to signal upward whether the leaf emptied (`_delete` entry, lines 39-47)
- It only frees empty leaves at **depth 2** when the child is **not leftmost** (lines 64-73 of the `_delete` entry)
- Empty leftmost leaves or empty leaves deeper than depth 2 are left allocated with zero keys permanently (`_delete` entry, line 113)

There is no merge, no redistribution, no tombstone marking, no deferred pass. The grep for `merge|redistribute|rebalance|underflow|borrow|sibling` across the entire codebase returns **zero matches** (underflow rebalancing entry, line 12).

## The Spectrum of Strategies

The underflow rebalancing entry (`topic-b-tree-underflow-rebalancing.md:42-57`) maps out how major databases handle this differently:

| Database | Strategy | Trade-off |
|----------|----------|-----------|
| **PostgreSQL** | Mark dead, VACUUM later, never merge pages | Max throughput, space degrades until VACUUM |
| **InnoDB** | Full merge on underflow + change buffer for batching | Close to textbook, amortized via write buffering |
| **SQLite** | Full merge/redistribute when page drops below ~1/3 full | Textbook correctness, feasible because single-writer |
| **This impl** | No merge, free only completely empty leaves at depth 2 | Simplest possible, space degrades permanently |

PostgreSQL sits at a pragmatic sweet spot: it doesn't pay the cost of immediate rebalancing *or* the cost of permanent space waste. The deferred VACUUM model means that:
- **Reads are never blocked** by structural maintenance
- **Deletes are fast** — just flag a tuple
- **Space is eventually reclaimed** — VACUUM handles it on its own schedule
- **But sparse pages accumulate** between VACUUM runs, degrading range scan I/O

The fill-factor impact is real. The comparison table in the underflow entry (lines 60-68) shows that without rebalancing, leaves can degrade to near-empty, inflating the number of page reads for range scans and wasting disk. PostgreSQL accepts this temporarily; the reference implementation accepts it permanently.

## The Structural Asymmetry

One of the sharpest observations across the entries is the **asymmetry between insert and delete** in the reference implementation:

- **`_insert`** fully propagates splits upward: when a node overflows, it splits, returns `(mid_key, new_page)`, and the caller inserts into the parent — cascading all the way to root creation if needed (`_insert` entry, lines 53-84)
- **`_delete`** swallows the `'empty'` signal after one level of cleanup: if it can't handle it at depth 2, it converts `'empty'` to `True` and the information is lost (`_delete` entry, lines 72-73)

This means the tree **height only increases, never decreases** (`topic-b-tree-underflow-rebalancing.md`, line 87). PostgreSQL has the same property — its B-tree height rarely shrinks — but it mitigates the space impact through VACUUM's page recycling.

## Why This Matters for DDIA

The PostgreSQL pattern illustrates a principle central to *Designing Data-Intensive Applications*: **production systems routinely trade strict invariants for operational characteristics.** The textbook B-tree invariant (minimum 50% fill) is correct but expensive to maintain under concurrency. PostgreSQL relaxes it in favor of a batch-oriented background process, accepting temporary space waste for consistent throughput. This is the same principle behind LSM-tree compaction, MVCC's deferred garbage collection, and many other "clean up later" patterns in real databases.

The reference implementation in `btree.py` demonstrates the *structural plumbing* — the sibling chain in `_serialize_leaf` (underflow rebalancing entry, line 71), the free list in `PageManager` (lines 85–101), the three-valued return from `_delete` — that a rebalancing or VACUUM-style pass would build on. The infrastructure exists; the policy doesn't. That makes it a useful teaching tool: you can see *what* would need to happen without the concurrency and correctness complexities that make the production version hard to follow.

---

## Topics to Explore

- [general] `postgres-nbtree-half-dead-state-machine` — PostgreSQL's two-phase page deletion (half-dead → dead) is the mechanism that makes deferred cleanup crash-safe; understanding it reveals why "just free the page" (as in `btree.py`) is insufficient for production
- [general] `concurrent-btree-deletion` — Blink-tree latch coupling during merge/delete is where the real complexity lives; the single-threaded reference implementation sidesteps it, but understanding it explains *why* PostgreSQL chose to defer
- [function] `b-tree-storage-engine/btree.py:_delete` — Trace the three-valued return path and the depth-2 cleanup guard to see exactly where the implementation draws the line on structural maintenance
- [function] `b-tree-storage-engine/btree.py:allocate_page` — The free list (lines 85–94) is where pages from merge or VACUUM would be recycled; understanding it is prerequisite to implementing either
- [file] `b-tree-storage-engine/test_btree.py` — `test_delete_frees_empty_leaf` (line 161+) tests the only cleanup path that exists; worth checking what edge cases it misses (leftmost child, depth > 2, partially-empty leaves)

## Beliefs

- `postgres-defers-btree-rebalancing-to-vacuum` — PostgreSQL's `nbtree` never merges or redistributes siblings on delete; it marks tuples dead and relies on VACUUM to reclaim space, trading immediate space efficiency for throughput under concurrency
- `btree-impl-asymmetric-split-vs-merge` — The reference implementation fully propagates splits upward through `_insert`'s three-valued return but swallows the `'empty'` signal in `_delete` after one level, making insertion structurally complete but deletion structurally incomplete
- `tree-height-monotonically-increases` — Since neither the reference implementation nor PostgreSQL merges internal nodes on delete, tree height only grows (on root splits) and never shrinks, even under heavy deletion
- `sibling-chain-enables-but-unused-for-rebalancing` — Leaf pages serialize `next_sibling` pointers for range scans, providing exactly the infrastructure merge/redistribute would need, but no delete code path reads siblings for rebalancing purposes
- `deferred-cleanup-is-production-pattern` — PostgreSQL, LSM-tree compaction, and MVCC garbage collection all follow the same principle: defer structural maintenance to a background process to avoid blocking the write path under concurrency

