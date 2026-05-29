# Topic: PostgreSQL's `nbtree` README documents their actual Blink-tree latch coupling protocol, showing the complexity this implementation avoids

**Date:** 2026-05-29
**Time:** 07:51

# PostgreSQL's Blink-tree Complexity vs. This Implementation's Simplicity

## The Core Contrast

PostgreSQL's `nbtree` implements the Lehman-Yao Blink-tree protocol, which solves a hard problem: **how do concurrent readers and writers share a B-tree without blocking each other?** The answer involves a careful latch coupling dance — acquiring a lock on a child node before releasing the parent's lock, handling the case where a page splits while you're descending, and following right-links to find keys that moved during a concurrent split.

This implementation sidesteps all of that. The grep for `lock|latch|mutex|concurrent|thread|Lock` across the entire codebase returns **zero matches**. There is no concurrency control whatsoever. The `BTree` class in `b-tree-storage-engine/btree.py` is a single-threaded, single-process storage engine.

## What PostgreSQL's Protocol Requires

The nbtree README documents several interacting mechanisms:

1. **Latch coupling (crabbing):** When descending the tree, you hold a latch on the parent while acquiring a latch on the child, then release the parent. This prevents the child from being split or deleted out from under you.

2. **Right-link chasing:** Every internal and leaf page has a right-link pointer to its sibling. If you descend to a page and discover the key you want has moved right (because a concurrent split happened between your parent read and child read), you follow the right-link. This is the core Lehman-Yao insight — it makes splits non-blocking for readers.

3. **Incomplete split handling:** A split might crash halfway through — the new right sibling exists and the old page's right-link points to it, but the parent hasn't been updated yet. Readers must detect and tolerate this. Writers encountering an incomplete split must finish it before proceeding.

4. **Page deletion protocol:** Deleting a leaf page requires a multi-step protocol: mark the page half-dead, update the left sibling's right-link, remove the downlink from the parent, then add the page to the free list. Each step needs specific latch ordering to avoid deadlocks.

## What This Implementation Does Instead

The `PageManager` class (`btree.py:27`) does direct synchronous I/O — `read_page` and `write_page` are blocking calls with no latch acquisition. The `allocate_page` method (`btree.py:84`) reads metadata, checks the free list, and updates the metadata in a single-threaded sequence with no atomicity concerns beyond the WAL.

Leaf pages do have a `next_sibling` pointer (serialized in `_serialize_leaf` with `struct.pack('>I', next_sibling)`), but this serves range scans — not concurrent split recovery. There's no right-link chasing logic because there's no concurrent split to race against.

The WAL (`btree.py:109`) provides crash safety but not concurrency. It logs page writes, replays them on recovery, and uses `fsync` for durability. The `fix-plan.md` documents real bugs in this WAL protocol (fsync ordering, metadata not logged, weak checksums), but notably **none of the six bugs are concurrency-related**. Every bug is about single-writer crash safety.

## Why This Matters Pedagogically

The DDIA reference implementations demonstrate storage engine concepts in isolation. This B-tree teaches page layout, split mechanics, WAL recovery, and free-list management without entangling those concepts with concurrent access control. PostgreSQL's nbtree README shows what a production system must add: the latch coupling protocol alone is arguably more complex than this entire 612-line implementation.

The gap between the two is where most of the engineering cost lives in real database systems — not in the data structure itself, but in making it safe for concurrent access.

## Topics to Explore

- [general] `postgres-nbtree-readme` — Read PostgreSQL's actual `src/backend/access/nbtree/README` to see the latch coupling protocol, incomplete-split handling, and page deletion dance documented in detail
- [function] `b-tree-storage-engine/btree.py:_serialize_leaf` — The `next_sibling` pointer here serves range scans; compare with how Blink-trees use right-links for concurrent split recovery
- [file] `b-tree-storage-engine/fix-plan.md` — All six bugs are single-writer crash safety issues, illustrating that even without concurrency the WAL protocol has subtle correctness requirements
- [general] `lehman-yao-1981` — The original Lehman & Yao paper "Efficient Locking for Concurrent Operations on B-Trees" that introduced right-links and the high-key optimization PostgreSQL builds on
- [function] `b-tree-storage-engine/btree.py:allocate_page` — Page allocation with free-list reuse; in a concurrent system this would need its own latch protocol to prevent two writers from allocating the same page

## Beliefs

- `no-concurrency-control` — The B-tree implementation contains zero locking, latching, or thread-safety primitives; it is strictly single-threaded single-process
- `sibling-links-for-scans-not-splits` — Leaf `next_sibling` pointers exist for range scan traversal, not for Blink-tree concurrent split recovery
- `all-bugs-are-crash-safety` — Every bug documented in `fix-plan.md` concerns single-writer WAL/fsync crash safety; none involve concurrent access races
- `wal-provides-durability-not-isolation` — The WAL ensures crash recovery (replaying page writes) but provides no transaction isolation or concurrent writer support

