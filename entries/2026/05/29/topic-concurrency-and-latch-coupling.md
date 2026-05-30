# Topic: How production databases (InnoDB, PostgreSQL) handle concurrent range scans and splits on linked leaves; this implementation is single-threaded, so it avoids the hardest part of the design

**Date:** 2026-05-29
**Time:** 10:42

# Concurrent Range Scans and Splits on Linked Leaves

## What This Implementation Does

The B-tree in `b-tree-storage-engine/btree.py` is a B+-tree with a singly-linked leaf chain. Every leaf carries a `next_sibling` page pointer (4 bytes, sentinel `NO_SIBLING = 0xFFFFFFFF` at line 14). When `range_scan` finds the first matching key, it walks this chain horizontally — one page read per leaf — without re-traversing internal nodes.

Splits maintain the chain: the new right page inherits the old leaf's `next_sibling`, and the old leaf rewires its pointer to the new right page. The WAL ensures crash-atomicity by logging the right page before the left page, so the pointer target is always durable before anything references it.

**None of this needs concurrency control because the implementation is single-threaded.** There is no latching, no lock coupling, no synchronization of any kind. `PageManager.read_page` (line 70) and `write_page` (line 77) operate on a bare file handle. `allocate_page` (line 88) does an unprotected read-modify-write on metadata. This is the correct design choice for a teaching implementation — but it dodges the hardest engineering problem in B-tree design.

## The Concurrent Problem: Scan Meets Split

The dangerous scenario is simple to state and brutal to solve:

1. **Thread A** is executing a `range_scan`. It has read leaf page L₁ and is about to follow L₁'s `next_sibling` pointer to page L₂.
2. **Thread B** inserts a key into L₂, causing L₂ to split into L₂ (lower keys) and L₃ (upper keys). Thread B rewrites L₂'s `next_sibling` to point to L₃, and L₃ inherits L₂'s old `next_sibling`.
3. **Thread A** resumes and reads L₂ — but L₂ now contains only the lower half of its original keys. Thread A follows the chain to L₃, L₄, etc. This is fine; no keys are lost.

But consider a worse case:

1. **Thread A** reads L₁, observes `next_sibling = L₂`.
2. **Thread B** splits L₁ itself. The lower half stays in L₁ (which Thread A already read), the upper half goes into a new page L₁'. L₁'s `next_sibling` now points to L₁', and L₁' points to L₂.
3. **Thread A** follows its cached pointer to L₂ — **skipping L₁' entirely**. Keys that were in the upper half of the original L₁ are silently missing from the scan results.

This is the phantom problem at the physical level: the structure changed under the reader's feet.

## How PostgreSQL Solves It: Lehman-Yao

PostgreSQL's nbtree uses the **Lehman-Yao (L&Y) algorithm**, which is specifically designed to minimize latching during concurrent access:

**Right-links on every node, not just leaves.** This implementation's `next_sibling` pointer in `_serialize_leaf` (line 178: `buf += struct.pack('>I', next_sibling)`) exists only on leaves. L&Y extends this to internal nodes. Every node has a right-link to its sibling at the same level.

**High keys.** Every node stores a **high key** — the upper bound of keys that belong in this node. This implementation has no high key. In L&Y, if a reader descends to a node and discovers its search key exceeds the high key, it knows a split occurred and follows the right-link to the new sibling.

**Single-latch protocol.** A reader latches only one node at a time. It reads a node, unlatches it, then latches the child. If a concurrent split moved keys to a right sibling between the unlatch and the child-latch, the reader detects this via the high key and follows the right-link. No restart needed. Compare this to this implementation, where `range_scan` can freely read `next_sibling` from one page and then read the target page with no concern for interleaving — because there is no interleaving.

**Split protocol.** When a node splits:
1. The new right sibling is created (latched exclusively).
2. The original node's right-link is updated to point to the new sibling.
3. Only then is the parent updated with a new separator key.

If a reader arrives between steps 2 and 3, the parent doesn't yet point to the right sibling — but the reader can still reach it via the right-link. The tree is always traversable, even in a "half-split" state. This is precisely the gap this implementation's WAL entry documents as `btree-partial-split-replay-orphans-right-page`: after a crash mid-split, point lookups fail for keys in the new right page because the parent hasn't been updated, but range scans via the sibling chain still find them. L&Y turns this "recovery artifact" into a deliberate runtime state.

## How InnoDB Solves It: Optimistic Lock Coupling

InnoDB takes a different approach centered on **latches per buffer pool page** (not per tree node — the distinction matters because InnoDB accesses pages through its buffer pool, not through raw file I/O like `PageManager.read_page`):

**Read-write latches (S/X).** Readers acquire shared (S) latches; writers acquire exclusive (X) latches. Multiple concurrent `range_scan` equivalents proceed without blocking each other.

**Optimistic coupling.** For writes, InnoDB first descends the tree acquiring only S-latches — optimistically assuming no split will be needed. If the leaf has room, the write proceeds with just an X-latch on the leaf. If a split is needed, the operation restarts from the root with X-latches on the path. Since most inserts don't cause splits (a leaf with max keys `N` is full only 1/N of the time), the optimistic path dominates.

**Index lock for structural modifications.** When a split or merge does happen, InnoDB acquires a higher-level "index lock" that prevents other structural modifications (but not reads) from proceeding concurrently.

**Gap locks for range scans.** For the logical isolation problem (phantoms in `SERIALIZABLE`/`REPEATABLE READ`), InnoDB uses **next-key locks** — a combination of a record lock and a gap lock on the interval before the next key. This prevents concurrent inserts into a range being scanned. This is a locking concern (transaction isolation), not a latching concern (thread safety) — a distinction this implementation collapses entirely by being single-threaded.

## Why This Is the Hardest Part

The leaf chain in `btree.py` is ~6 lines of code: a 4-byte pointer in the serialization format, a parameter threaded through split logic, and a while loop in `range_scan`. The concurrent version of the same feature requires:

| Concern | This implementation | PostgreSQL (L&Y) | InnoDB |
|---------|-------------------|-------------------|--------|
| Right-links | Leaves only | All nodes | N/A (uses lock coupling) |
| High keys | None | Every node | N/A |
| Latches per operation | None | 1 at a time | 2 (parent + child) |
| Split visibility | Atomic (single-threaded) | Half-splits are legal states | Protected by index lock |
| Range scan isolation | N/A | Predicate locks (SSI) | Next-key locks |
| Buffer pool integration | None (`read_page` hits file) | Shared buffers with pinning | Buffer pool with LRU |

The leaf chain is elegant precisely because it avoids all of this. The `next_sibling` field at line 14's `NO_SIBLING = 0xFFFFFFFF` is a finished design for sequential access — but only in a world where no one else is modifying the chain while you're walking it.

## Topics to Explore

- [general] `lehman-yao-high-keys` — Adding a high key to each node is the mechanism that lets L&Y readers detect concurrent splits without holding parent latches; trace how this differs from the simple `next_sibling` in `_serialize_leaf`
- [general] `innodb-next-key-locks` — InnoDB's next-key locking prevents phantom reads during range scans at the logical (transaction) level, separate from the physical latching that prevents torn reads
- [function] `b-tree-storage-engine/btree.py:allocate_page` — The unprotected read-modify-write at lines 88-96 is the smallest example of why concurrency is hard here; two concurrent splits would allocate the same page number
- [general] `buffer-pool-latch-integration` — Production B-trees latch buffer pool frames, not pages on disk; understanding this layer explains why `PageManager.read_page` is unsuitable for concurrent access
- [file] `write-ahead-log/wal.py` — The standalone WAL has `threading.Lock` at line 80, making it the only concurrency-aware component; compare its `append_batch` with InnoDB's group commit to see how WAL serialization becomes a bottleneck

## Beliefs

- `btree-no-concurrency-control` — `BTree` in `btree.py` has no latches, mutexes, or synchronization; concurrent access to any method is unsafe and would corrupt the tree
- `leaf-chain-correct-single-threaded` — The `next_sibling` leaf chain is correctly maintained through splits via WAL write ordering, but only because no concurrent reader can observe intermediate states
- `lehman-yao-needs-high-keys-and-internal-links` — The existing leaf `next_sibling` pointer is necessary but insufficient for Lehman-Yao; the algorithm also requires right-links on internal nodes and a high key per node, neither of which this implementation has
- `concurrent-scan-split-can-skip-keys` — Without latching, a range scan that caches a `next_sibling` pointer can miss keys moved to a new sibling page by a concurrent split — the core phantom problem at the physical level

