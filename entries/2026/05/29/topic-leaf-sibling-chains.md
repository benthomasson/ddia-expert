# Topic: B+ tree leaf linking is a design choice not present in classic B-trees; explore why DDIA implementations use it and the tradeoff vs. re-descending from the root.

**Date:** 2026-05-29
**Time:** 07:45

I have enough from the observations to give a thorough explanation. Let me work with what's there.

---

# B+ Tree Leaf Linking in the DDIA Implementation

## The Design Choice

The B-tree implementation in `b-tree-storage-engine/btree.py` is actually a **B+ tree** — leaves are linked forward via a `next_sibling` pointer. This is visible in three places:

1. **The sentinel constant** (line 13): `NO_SIBLING = 0xFFFFFFFF` — a dedicated marker for "no next leaf."

2. **The leaf page format** — `_serialize_leaf()` (starting ~line 182) takes `next_sibling` as a parameter and writes it immediately after the page header:
   ```python
   def _serialize_leaf(keys, values, next_sibling=NO_SIBLING):
       buf = struct.pack(HEADER_FMT, LEAF, len(keys))
       buf += struct.pack('>I', next_sibling)
   ```

3. **`_deserialize_leaf()`** (starting ~line 193) reads back `next_sib` from the same offset, returning it as part of the tuple `(keys, values, next_sibling)`.

4. **`_write_empty_leaf()`** (line 48) initializes every new leaf with `NO_SIBLING`, establishing the invariant that the rightmost leaf in the tree has no forward link.

The `range_scan` method at line 486 is what *consumes* these links. The tests confirm the behavior: `test_range_scan` (line 77) inserts 20 keys across multiple leaves (with `max_keys_per_page=4`, that's at least 5 leaves), then scans from `"k05"` to `"k10"` and expects `["k05", "k06", "k07", "k08", "k09"]` — keys that span multiple leaf pages.

## Why Leaf Linking, Not Re-Descending

A classic B-tree stores data in both internal and leaf nodes and has no sibling pointers. To scan a range, you'd descend from the root for each key (or at least for each new leaf). The cost of re-descending is **O(log n) per leaf visited**.

Leaf linking converts range scans into a **single descent + sequential leaf walk**:

1. Descend from root to find the leaf containing `start_key` — cost: O(log n) page reads.
2. Walk `next_sibling` pointers until you pass `end_key` — cost: O(k/B) page reads, where k is the number of results and B is keys per leaf.

The tradeoff:

| | Leaf-linked (B+ tree) | Re-descend (classic B-tree) |
|---|---|---|
| **Range scan I/O** | O(log n + k/B) | O((k/B) × log n) |
| **Point lookup** | Same — O(log n) | Same — O(log n) |
| **Write cost** | Slightly higher — splits must update the previous leaf's `next_sibling` pointer | Simpler splits |
| **Page format** | 4 extra bytes per leaf for the pointer | No overhead |
| **Sequential access** | Excellent — leaves form a logical linked list matching sort order | Must re-navigate the tree structure |

The test `test_page_io_counts` (line 95) validates that point lookups read at most `height + 1` pages. With leaf linking, a range scan touching 3 leaves reads `height + 3` pages total instead of `3 × height`.

## The Tradeoff in Practice

The 4-byte `next_sibling` cost per leaf page is negligible — on a 4096-byte page, it's <0.1% overhead. The write-amplification cost during splits is real but modest: when a leaf splits, you allocate a new page, set the new page's `next_sibling` to the old page's `next_sibling`, then update the old page's `next_sibling` to point to the new page. This is two extra field writes during an operation that already rewrites both pages.

The **real cost** is architectural: leaf linking introduces a horizontal dependency between pages that doesn't exist in a pure tree. This matters for:

- **Concurrency**: A range scan holds a "cursor" that follows sibling pointers, which can conflict with concurrent splits. Real databases (PostgreSQL, InnoDB) solve this with page-level locking protocols or latch-coupling.
- **Recovery complexity**: The WAL (lines 113–176) must ensure that split operations update the sibling pointer atomically with the page split. If a crash happens between writing the new leaf and updating the old leaf's pointer, the chain is broken.

This implementation uses WAL-based crash safety (`WAL.log_write` at line 131, `WAL.recover` at line 142) to handle the atomicity requirement — all page writes in a split are logged before any are applied.

## Why DDIA Implementations Use It

Kleppmann's DDIA describes B-trees as the dominant on-disk index structure, and virtually all production B-tree implementations (InnoDB, PostgreSQL, SQLite, LMDB) are actually B+ trees with leaf linking. The DDIA implementation follows this pattern because:

1. **Range queries are a first-class operation** — as shown by `range_scan` and the full iteration support (`__iter__`), the tree is designed for ordered access, not just point lookups.
2. **It mirrors real database storage engines** — the implementation is pedagogical, meant to demonstrate how production systems work. A B-tree without leaf linking would be academically simpler but wouldn't teach the reader how InnoDB or PostgreSQL actually store data.
3. **The LSM tree comparison** — the repo also contains `log-structured-merge-tree/lsm.py` with its own `range_scan` (line 272). Having both storage engines support range scans with comparable interfaces lets the reader compare the two approaches from DDIA Chapter 3 directly.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:range_scan` — The consumer of leaf linking; understanding how it walks sibling pointers reveals the payoff of the design
- [function] `b-tree-storage-engine/btree.py:_serialize_leaf` — Shows exactly how the 4-byte sibling pointer is packed into the page format alongside keys and values
- [general] `split-and-sibling-update-atomicity` — How the WAL ensures that a leaf split and its sibling pointer update are crash-atomic; a broken chain would silently truncate range scans
- [file] `log-structured-merge-tree/lsm.py` — Compare range scan implementation: LSM trees get sequential reads "for free" from sorted SSTables, while B+ trees need explicit leaf linking to achieve the same
- [general] `concurrency-and-latch-coupling` — How production databases (InnoDB, PostgreSQL) handle concurrent range scans and splits on linked leaves; this implementation is single-threaded, so it avoids the hardest part of the design

## Beliefs

- `btree-leaf-pages-have-forward-sibling-pointer` — Every leaf page in `btree.py` contains a 4-byte `next_sibling` field (immediately after the 3-byte header), set to `NO_SIBLING` (0xFFFFFFFF) for the rightmost leaf
- `range-scan-walks-sibling-chain` — `range_scan` (line 486) descends to the starting leaf once, then follows `next_sibling` pointers horizontally rather than re-descending from the root for each subsequent leaf
- `leaf-linking-cost-is-split-maintenance` — When a leaf splits, both the old and new page must be written: the new page inherits the old page's `next_sibling`, and the old page's `next_sibling` is updated to point to the new page
- `wal-protects-sibling-pointer-consistency` — The WAL logs all page writes before applying them, ensuring that a crash during a leaf split cannot leave a broken sibling chain that would silently truncate range scans

