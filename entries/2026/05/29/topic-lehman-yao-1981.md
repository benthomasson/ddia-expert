# Topic: The original Lehman & Yao paper "Efficient Locking for Concurrent Operations on B-Trees" that introduced right-links and the high-key optimization PostgreSQL builds on

**Date:** 2026-05-29
**Time:** 10:56

# Lehman & Yao's B-link Trees and This Implementation

## What the Paper Introduced

Lehman & Yao (1981) solved a fundamental problem: how do you let multiple threads search and insert into a B-tree concurrently without locking the entire tree? Their two key innovations were:

1. **Right-link pointers on every node** (not just leaves) — each node carries a pointer to its right sibling at the same level. If a searcher arrives at a node that has just been split, the right-link lets it "move right" to find the key that migrated to the new node during the split.

2. **High-key** — each node stores the upper bound of keys it can contain. A searcher compares its target against the high-key to detect whether it needs to follow the right-link. If `target > high_key`, the node was split and the key lives to the right.

Together, these let you use **lock coupling** (hold at most two latches at a time) instead of locking an entire root-to-leaf path, which is the foundation PostgreSQL's `nbtree` builds on.

## What This Codebase Has — and Doesn't

The implementation at `b-tree-storage-engine/btree.py` has **leaf-level sibling links** but is **not a B-link tree**:

**Present: Leaf sibling pointers.** The `_serialize_leaf` function (line ~170) writes a `next_sibling` field into every leaf page:

```python
def _serialize_leaf(keys, values, next_sibling=NO_SIBLING):
    buf = struct.pack(HEADER_FMT, LEAF, len(keys))
    buf += struct.pack('>I', next_sibling)
```

The sentinel `NO_SIBLING = 0xFFFFFFFF` (line 13) marks the rightmost leaf. These links enable range scans (tested in `test_btree.py` lines ~80–90) by walking leaf-to-leaf without re-traversing from the root.

**Missing: Right-links on internal nodes.** Lehman & Yao require right-links at *every* level so a concurrent searcher can recover from a split at any point in the tree. This implementation only links leaves.

**Missing: High-keys.** No node stores an upper-bound fence key. Without high-keys, a searcher can't detect "this node was split and my key moved right." The grep for `high.?key|fence|upper.?bound` returned zero matches.

**Missing: Any concurrency control.** The grep for `latch|lock|concurrent|thread|mutex` returned zero matches. This is a single-threaded B-tree — the entire Lehman & Yao protocol is irrelevant without concurrent access. The right-link and high-key optimizations exist specifically to *minimize locking*, so they only matter when there are locks to minimize.

## Why It Matters

The leaf sibling links in this code serve a different purpose than Lehman & Yao's right-links. Here they're a **scan optimization** — follow the chain for `range_scan` instead of re-descending. In Lehman & Yao, they're a **concurrency correctness mechanism** — follow the link to find keys that moved during a concurrent split.

This is the critical distinction: the same structural feature (a pointer to the right sibling) serves completely different roles depending on whether the tree supports concurrent access.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_serialize_leaf` — Leaf serialization showing the sibling pointer format; compare with `_serialize_internal` to confirm internal nodes lack right-links
- [general] `postgres-nbtree-high-key` — PostgreSQL's `nbtree` README documents how it extends Lehman & Yao with high-keys stored as the first item in each page
- [general] `latch-coupling-protocol` — The lock-coupling (crabbing) technique that B-link trees enable: hold parent latch only until child latch is acquired
- [function] `b-tree-storage-engine/btree.py:_search` — How search traverses the tree without right-link recovery; adding concurrency would require a "move right" step here
- [general] `split-half-completed-visibility` — The core scenario Lehman & Yao solve: what a concurrent reader sees when it arrives at a node mid-split

## Beliefs

- `leaf-sibling-not-blink` — Leaf `next_sibling` pointers serve range scans, not Lehman & Yao concurrent-split recovery; internal nodes have no right-links
- `no-concurrency-control` — The B-tree implementation is entirely single-threaded with no latching, locking, or thread-safety mechanisms
- `no-high-key-fence` — No node stores a high-key (upper bound); searchers cannot detect mid-split state, which would be required for a true B-link tree
- `sibling-sentinel-convention` — `NO_SIBLING = 0xFFFFFFFF` marks the rightmost leaf at each level; this is the only sibling metadata stored per node

