# Topic: Lehman and Yao's 1981 paper "Efficient Locking for Concurrent Operations on B-Trees" defines the sibling-link protocol this implementation alludes to but doesn't need

**Date:** 2026-05-29
**Time:** 07:51

## The Lehman-Yao Connection: Sibling Links Without Concurrency

This B-tree implementation contains the **structural signature** of the Lehman-Yao (L&Y) B-link tree — sibling pointers between nodes — but uses them for an entirely different purpose than the original paper intended.

### What the implementation has

Every leaf page carries a `next_sibling` pointer, written as a 4-byte big-endian integer immediately after the page header. You can see the sentinel value defined at `b-tree-storage-engine/btree.py:13`:

```python
NO_SIBLING = 0xFFFFFFFF
```

The pointer is baked into the page format at two points:
- **`_write_empty_leaf`** (line ~51) writes `NO_SIBLING` as the initial sibling pointer for every new leaf
- **`_serialize_leaf`** (line ~167) accepts `next_sibling` as a parameter and encodes it right after the header

These links form a singly-linked chain across all leaf pages at the same level, enabling sequential range scans — the test at line ~82 of `test_btree.py` exercises exactly this: `tree.range_scan("k05", "k10")` walks the leaf chain without re-traversing from the root.

### What the Lehman-Yao paper actually solves

Lehman and Yao's insight was about **concurrency**, not range scans. In a concurrent B-tree, a searcher descending the tree can be invalidated mid-traversal by a splitter modifying a node the searcher already passed through. The L&Y protocol adds a right-link (sibling pointer) to **every** node — internal nodes included — so a searcher that lands on a node post-split can follow the link rightward to find the key that was moved, without acquiring a read lock on the parent. This reduces locking to at most one node at a time (latch coupling), which is the paper's main contribution.

### Why this implementation doesn't need it

The grep results tell the full story: searching for `thread`, `concurrent`, `latch`, `lock`, and `mutex` across the codebase yields **zero matches**. This is a single-threaded, single-process B-tree. There is no concurrent access, so there is no split-traversal race to protect against. The sibling links exist purely as a convenience for leaf-level iteration (range scans), which is a feature of nearly every B+ tree implementation regardless of concurrency model.

Put differently: the implementation borrows the **data structure** (linked leaves) but has no need for the **protocol** (latch-free descent via right-links on internal nodes). Internal nodes in this implementation almost certainly lack sibling pointers entirely — the serialization function shown is `_serialize_leaf`, and the sibling field is specific to the leaf format.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_serialize_leaf` — Trace how `next_sibling` is set during leaf splits to understand when the chain gets wired up
- [general] `internal-node-format` — Check whether internal nodes also carry sibling pointers (they shouldn't need to in a single-threaded implementation, confirming the L&Y distinction)
- [general] `range-scan-implementation` — Follow the range scan code path to see how it walks the leaf chain using `next_sibling`, which is the actual use of the sibling links
- [general] `split-during-insert` — Examine how leaf splits maintain the sibling chain invariant (the new leaf must be linked in correctly or range scans break)
- [file] `b-tree-storage-engine/btree.py` — Read the full BTree class to see if there is any concurrency-aware code elsewhere (the grep says no, but worth confirming)

## Beliefs

- `leaf-pages-carry-sibling-pointers` — Every leaf page stores a 4-byte `next_sibling` pointer immediately after the 3-byte header, forming a singly-linked chain across all leaves
- `no-concurrency-control-exists` — The implementation contains zero locking, latching, or threading primitives; it is strictly single-threaded single-process
- `sibling-links-serve-range-scans-not-concurrency` — The `next_sibling` pointers enable sequential leaf traversal for range queries, not Lehman-Yao style lock-free concurrent descent
- `no-sibling-sentinel-is-max-uint32` — `NO_SIBLING = 0xFFFFFFFF` marks the end of the leaf chain; a leaf with this value is the rightmost at its level

