# Topic: This implementation has no locking — explore how production B-trees (e.g., InnoDB, PostgreSQL) use latching and lock coupling for concurrent access

**Date:** 2026-05-29
**Time:** 10:25

# No Locking in This B-Tree — How Production Systems Do It

## The Gap

This B-tree implementation (`b-tree-storage-engine/btree.py`) is entirely single-threaded. `PageManager.read_page` (line 70) and `write_page` (line 77) operate directly on a single file handle with no synchronization. `allocate_page` (line 88) reads metadata, computes the next free page, and writes it back — a classic read-modify-write sequence with no atomicity guarantee. If two threads called `allocate_page` concurrently, both could allocate the same page number.

The standalone `write-ahead-log/wal.py` does use a `threading.Lock` (line 80: `self._lock = threading.Lock()`), and every mutating method (`append` at line 139, `append_batch` at line 150, `checkpoint` at line 163) acquires it. But this protects only the WAL's internal state — it doesn't coordinate B-tree structural modifications like splits.

This is fine for a teaching implementation. Production systems need a very different approach.

## Latching vs. Locking

Production databases distinguish two concepts that this codebase collapses:

- **Locks** protect logical data (rows, key ranges, predicates) for transaction isolation. They're held for the duration of a transaction — potentially seconds or minutes. InnoDB's row-level locks and PostgreSQL's `SELECT FOR UPDATE` are locks.

- **Latches** (sometimes called "mutexes" or "lightweight locks") protect physical data structures (buffer pool pages, B-tree nodes) for thread safety. They're held for microseconds — just long enough to read or modify an in-memory page. The `threading.Lock()` in `wal.py:80` is closer to a latch in spirit, though it protects the WAL rather than tree pages.

The B-tree in `btree.py` needs latches, not locks. The problem isn't transaction isolation — it's that two threads traversing the tree simultaneously could see a half-written page or a node mid-split.

## Lock Coupling (Crabbing)

The naive approach — latch the entire tree for every operation — kills concurrency. Production systems use **lock coupling** (also called "crabbing"):

1. **Latch the parent** before descending.
2. **Latch the child** page.
3. **Release the parent** if the child is "safe" (won't split on insert, won't merge on delete).
4. Repeat down the tree.

Consider a lookup in this implementation's tree. `BTree.get` would traverse from root to leaf, calling `read_page` at each level (the test at line 95 asserts reads ≤ height + 1). With lock coupling, the traversal would hold at most two latches at any time — the current node and its parent — releasing the parent once the child is confirmed safe.

For writes, "safe" means the child node has room for an insertion without splitting (i.e., `num_keys < max_keys_per_page`). If the child might split, you must hold the parent's latch because the split will insert a new separator key into the parent.

### Why This Implementation Would Break

Look at what happens during a split in `btree.py`. The code must:
1. Read the full node (`read_page`)
2. Allocate a new sibling page (`allocate_page`, line 88)
3. Write the split contents to both pages (`write_page`)
4. Insert the separator key into the parent
5. Possibly split the parent too, cascading up to the root

Without latching, a concurrent reader could see the original node after step 3 but before step 4 — the sibling exists but the parent doesn't point to it yet. Keys in the new sibling become invisible. Worse, a concurrent writer might follow a stale pointer and corrupt data.

## How InnoDB Does It

InnoDB uses a combination of techniques:

- **Read-write latches** on each buffer pool page. Readers acquire shared (S) latches; writers acquire exclusive (X) latches. Multiple readers proceed concurrently.
- **Optimistic lock coupling**: on the way down, acquire only S-latches (even for writes). If the leaf turns out to need a split, restart the traversal with X-latches from the point where the tree might change. Most inserts don't cause splits, so the optimistic path is the common case.
- **The index lock**: a higher-level latch that protects structural modifications (splits, merges). Only acquired when the tree structure actually changes — not on every write.

## How PostgreSQL Does It

PostgreSQL's nbtree uses the **Lehman-Yao algorithm**, which is specifically designed to reduce latching:

- Every internal node has a **right-link pointer** to its sibling — similar to the `next_sibling` field in `_serialize_leaf` (`btree.py`, line 169: `buf += struct.pack('>I', next_sibling)`). But Lehman-Yao extends this to internal nodes too, not just leaves.
- A reader traversing down the tree might land on a node that has already been split. Instead of holding the parent latch to prevent this, the reader detects the split (the search key is beyond the node's high key) and follows the right-link.
- This means readers **never hold more than one latch** at a time. They latch a node, read it, unlatch it, then latch the child. If a concurrent split happened, they follow the right-link — no restart needed.

The `next_sibling` field already present in this implementation's leaf serialization is the seed of this idea, but Lehman-Yao requires a **high key** (the upper bound of keys in each node) and right-links on internal nodes, neither of which this implementation has.

## The WAL Interaction

The `WAL` class in `btree.py` (line 122) logs page writes before applying them. In a concurrent system, the WAL and latching must coordinate carefully:

- A page's latch must be held while writing to the WAL and modifying the page, ensuring the WAL entry reflects the actual page state.
- The WAL in `wal.py` serializes all writes through `self._lock` (line 139), which effectively makes the WAL a single-writer bottleneck. Production WALs use **group commit** — batching multiple transactions' WAL records into a single fsync — to amortize this cost. The `append_batch` method (line 150) hints at this pattern but applies it within a single transaction rather than across transactions.

## Topics to Explore

- [general] `lehman-yao-right-links` — The Lehman-Yao algorithm eliminates parent-to-child latch coupling by adding right-link pointers and high keys to every node; compare with the existing leaf `next_sibling` field at `btree.py:169`
- [function] `b-tree-storage-engine/btree.py:allocate_page` — This read-modify-write on metadata (lines 88-96) is the most obvious concurrency hazard; adding a latch here reveals the broader page-allocation coordination problem
- [general] `optimistic-vs-pessimistic-coupling` — InnoDB's optimistic approach (S-latch down, restart with X-latch on split) vs. pessimistic (X-latch from root on every write) is a key performance tradeoff worth studying
- [function] `write-ahead-log/wal.py:append_batch` — Group commit across transactions (not just within one) is how production WALs avoid making the WAL lock a throughput bottleneck
- [general] `buffer-pool-integration` — Production B-trees don't call `read_page`/`write_page` directly on files; they go through a buffer pool with pinning and replacement policies, and latches live on the buffer frame, not the page

## Beliefs

- `btree-no-concurrency-control` — `BTree` in `btree.py` has no latches, mutexes, or synchronization of any kind; concurrent access to any `BTree` method is unsafe
- `wal-has-threading-lock` — `WriteAheadLog` in `wal.py:80` uses a `threading.Lock` to serialize `append`, `append_batch`, `checkpoint`, and `truncate`, making it the only concurrency-aware component in the codebase
- `leaf-next-sibling-exists` — Leaf nodes store a `next_sibling` pointer (`btree.py:169`), which is a prerequisite for Lehman-Yao but insufficient without internal-node right-links and high keys
- `allocate-page-race-condition` — `PageManager.allocate_page` (lines 88-96) performs an unprotected read-modify-write on the metadata page; two concurrent callers can allocate the same page number

