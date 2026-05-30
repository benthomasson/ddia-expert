# Topic: Why B-link trees maintain sibling pointers and how their invariants interact with concurrent access and crash recovery

**Date:** 2026-05-29
**Time:** 10:48

# Why B-link Trees Maintain Sibling Pointers

## The Sibling Pointer in This Implementation

Every leaf node in `b-tree-storage-engine/btree.py` carries a 4-byte `next_sibling` field — a page number pointing to the leaf's right neighbor. The sentinel `NO_SIBLING = 0xFFFFFFFF` (line 13) marks the rightmost leaf. You can see this baked into the binary format at two levels:

- **On creation:** `PageManager._write_empty_leaf` (line 48–52) writes the header followed immediately by `struct.pack('>I', NO_SIBLING)`.
- **On serialization:** `_serialize_leaf` (line 182–190) takes `next_sibling` as a parameter and packs it right after the page header.
- **On deserialization:** `_deserialize_leaf` (line 193–194) returns a three-tuple `(keys, values, next_sibling)` — the pointer is always available to callers.

Internal nodes do **not** have sibling pointers. They store only routing keys and child page numbers. This asymmetry — sibling pointers on leaves only — is the B+-tree design, not a full B-link tree.

## What Sibling Pointers Buy You

The primary payoff is **range scan efficiency**. Without a leaf chain, scanning keys `"cherry"` through `"grape"` requires repeated tree traversals — descend, visit, ascend, descend again — touching O(N × h) pages for N results in a tree of height h.

With the sibling chain, the scan descends once to the first matching leaf (h page reads), then walks the linked list horizontally. Each subsequent leaf costs one page read. For N results spanning L leaves, the total is h + L reads, and the L reads are sequential if leaves were allocated in order.

The test at `test_btree.py:82–86` exercises this directly: a `range_scan("k05", "k10")` on a tree with `max_keys_per_page=4` must span multiple leaf pages, and it produces the correct ordered result without re-traversing internal nodes. The full-table iteration (`__iter__`) works the same way — descend to the leftmost leaf, then walk the chain to the end.

## How Splits Maintain the Chain

When a leaf overflows and splits, the sibling pointers must be rewired:

1. The new right page inherits the old leaf's `next_sibling` pointer.
2. The old leaf's `next_sibling` is rewritten to point to the new right page.

This preserves the linked-list invariant: the chain L₁ → L₂ → L₃ becomes L₁ → L₁' → L₂ → L₃ after splitting L₁. The cost is one extra page write per split — trivial compared to the page allocations and parent updates the split already requires.

## Crash Recovery and the WAL

The `WAL` class (lines 122–180) logs page writes before they're applied to the data file. Each entry carries a sequence number, page number, data payload, and a CRC32 checksum. On recovery, `WAL.recover` (line 144) replays all valid entries by rewriting pages, then truncates the WAL.

The interaction between sibling pointers and crash recovery is subtle. A split involves multiple page writes:

1. Write the new right sibling page (with its inherited `next_sibling`).
2. Rewrite the original leaf page (with its updated `next_sibling` pointing to the new sibling).
3. Update the parent internal node with the new separator key and child pointer.
4. Possibly update metadata (root pointer, if the root split).

If the process crashes mid-split, the WAL provides atomicity — but only for pages that were logged. The `fix-plan.md` documents that **metadata writes bypass the WAL entirely** (Bug 3), and the **WAL commit sequence is wrong** (Bug 2: it truncates the WAL before ensuring the data file is durable). This means a crash after a root split could leave the metadata pointing to a new root whose pages weren't fsynced.

Even with a correct WAL, a crash between steps 2 and 3 creates an interesting state: the sibling chain is intact (the new page is reachable by walking right from the old leaf), but the parent doesn't point to the new page. **Range scans still work** — they follow the chain — but **point lookups for keys in the orphaned right page fail** because the tree traversal can't reach it through the parent. This "half-split" state is exactly what the Lehman-Yao algorithm turns from a recovery artifact into a deliberate runtime invariant.

## The Gap: From B+-tree to B-link Tree

This implementation has the seed of B-link tree design — the leaf `next_sibling` pointer — but lacks two critical features that Lehman-Yao requires:

**Right-links on internal nodes.** In a B-link tree, *every* node (not just leaves) has a sibling pointer. This lets a reader who arrives at an internal node after a concurrent split follow the right-link to find the keys that moved, rather than requiring the parent to already be updated.

**High keys.** Each node stores an upper bound — the highest key that belongs in this node. When a reader descends to a child and finds its search key exceeds the high key, it knows a split occurred and follows the right-link. Without high keys, a reader has no way to detect that it's in the wrong node.

Together, these two additions enable the **single-latch protocol**: a reader latches only one node at a time, reads it, unlatches, then latches the child. If a concurrent split happened between unlatch and child-latch, the reader detects it via the high key and follows the right-link. No restart, no parent latch held. Compare this to lock coupling (crabbing), where the parent latch is held until the child is confirmed "safe."

## Why This Implementation Doesn't Need Any of It

`BTree` in `btree.py` is entirely single-threaded. There are no latches, no mutexes, no synchronization of any kind. `PageManager.read_page` (line 70) and `write_page` (line 77) operate on a bare file handle. `allocate_page` (lines 88–96) performs an unprotected read-modify-write on the metadata page — two concurrent callers would allocate the same page number.

This is the correct design choice for a teaching implementation. The sibling chain is maintained correctly through splits because no concurrent reader can observe intermediate states. The WAL provides crash atomicity (modulo the bugs in `fix-plan.md`). The result is a clean illustration of the B+-tree leaf chain — the mechanism that makes range queries O(h + L) — without the engineering complexity of making it work under concurrent access.

The standalone WAL module at `write-ahead-log/wal.py` is the only concurrency-aware component in the codebase: it uses `threading.Lock` (line 80) to serialize `append`, `append_batch`, `checkpoint`, and `truncate`. But this protects only the WAL's internal state, not the B-tree structure.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:range_scan` — Trace how the leaf sibling chain is actually walked; this is where the O(h + L) scan cost manifests and where a concurrent split would cause missed keys
- [general] `lehman-yao-high-keys` — Adding a high key to each node lets readers detect concurrent splits without holding parent latches; the key mechanism that separates B-link trees from B+-trees with sibling pointers
- [file] `b-tree-storage-engine/fix-plan.md` — Documents six bugs including metadata writes bypassing the WAL (Bug 3) and a WAL commit/truncate race (Bug 2); both directly affect crash recovery of split operations
- [general] `half-split-reachability` — After a crash mid-split, keys in the orphaned right page are reachable via the sibling chain but not via tree descent; Lehman-Yao makes this a legal runtime state, not just a recovery artifact
- [function] `write-ahead-log/wal.py:append_batch` — Group commit within a transaction; compare with cross-transaction group commit (InnoDB) to see how WAL serialization becomes a concurrency bottleneck

## Beliefs

- `sibling-pointers-leaf-only` — Sibling pointers exist only on leaf nodes (`_serialize_leaf` at line 182); internal nodes have no right-links, making this a B+-tree, not a B-link tree
- `no-high-key-per-node` — Nodes store no upper-bound "high key," so a reader cannot detect that a concurrent split moved keys to a right sibling; this is the second missing prerequisite for Lehman-Yao
- `wal-does-not-log-metadata` — `BTree._write_meta` calls `PageManager.write_meta` directly, bypassing the WAL; a crash after a root-pointer update but before data pages are durable corrupts the tree (`fix-plan.md` Bug 3)
- `split-chain-rewiring-order` — During a leaf split, the new right page is written with the old leaf's `next_sibling` before the old leaf is rewritten to point to the new page; WAL ordering makes the pointer target durable before anything references it
- `single-threaded-by-design` — `BTree` has zero synchronization; the sibling chain, split logic, and WAL interaction are correct only because no concurrent reader or writer can observe intermediate states

