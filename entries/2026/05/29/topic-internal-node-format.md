# Topic: Check whether internal nodes also carry sibling pointers (they shouldn't need to in a single-threaded implementation, confirming the L&Y distinction)

**Date:** 2026-05-29
**Time:** 10:52

## Internal Nodes Do Not Carry Sibling Pointers — Confirming the L&Y Distinction

The answer is definitive: **internal nodes have no sibling pointer; only leaf nodes do.** This is visible directly in the serialization formats.

### Leaf layout (from `_serialize_leaf`, `btree.py:~176`)

```
[type:1B][num_keys:2B][next_sibling:4B] ([key_len:2B][key][val_len:2B][val])...
```

The `next_sibling` field is a 4-byte page number written immediately after the 3-byte header (line ~178 in the observation):

```python
buf += struct.pack('>I', next_sibling)
```

The sentinel `NO_SIBLING = 0xFFFFFFFF` (line 13) marks the rightmost leaf.

### Internal layout (from `_serialize_internal`)

```
[type:1B][num_keys:2B][child₀:4B] ([key_len:2B][key][child:4B])...
```

The 4 bytes after the header are **`child₀`** — the leftmost child pointer — not a sibling link. There is no sibling field anywhere in the internal node format. The entry on `_serialize_internal` confirms it takes only `(keys, children)` as parameters, with no sibling argument. Similarly, `_deserialize_internal` returns only `(keys, children)` — a 2-tuple, versus the 3-tuple `(keys, values, next_sibling)` from `_deserialize_leaf`.

### Why this confirms the Lehman & Yao distinction

In Lehman and Yao's B-link tree (1981), **every** node — internal and leaf — carries a right-link (sibling pointer) plus a high key. This is essential for correctness under concurrent access: when a reader descends to a node that has been split by a concurrent writer, the right-link lets the reader move sideways to the correct half without re-traversing from the root.

This implementation is single-threaded. No concurrent reader can observe a half-finished split, so internal nodes don't need sideways navigation. Leaf nodes retain sibling pointers for a different reason entirely: they enable efficient `range_scan` and iteration (`__iter__`) by chasing the leaf chain without re-traversing the tree from the root for each successor.

The structural asymmetry — siblings on leaves but not on internals — is the hallmark of a classic B⁺-tree rather than a B-link tree.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_serialize_leaf` — Compare the leaf format (with `next_sibling` parameter) directly against `_serialize_internal` to see the structural divergence
- [function] `b-tree-storage-engine/btree.py:BTree._insert` — The split logic that sets sibling pointers on new leaf pages but not on new internal pages; this is where the asymmetry originates
- [function] `b-tree-storage-engine/btree.py:range_scan` — The consumer of leaf sibling pointers; shows why leaves need them even without concurrency
- [general] `btree-concurrency-control` — Explore what changes would be needed to add Lehman-Yao concurrency: right-links on internals, high keys, and latch coupling
- [general] `b-plus-tree-leaf-chains` — Deeper look at why B⁺-trees universally chain leaves but classical B-trees (which store values in internal nodes too) do not

---

## Beliefs

- `internal-nodes-no-sibling-pointer` — Internal nodes have no sibling pointer field; their binary layout is `[type:1B][num_keys:2B][child₀:4B]([keylen:2B][key][child:4B])...` with no room for a right-link
- `leaf-sibling-pointer-present` — Leaf nodes carry a 4-byte `next_sibling` field immediately after the header, using sentinel `NO_SIBLING = 0xFFFFFFFF` for the rightmost leaf
- `not-lehman-yao-blink-tree` — The tree is a classical single-threaded B⁺-tree, not a Lehman-Yao B-link tree, because internal nodes lack right-link pointers and there are no high keys or latches
- `sibling-asymmetry-reflects-purpose` — Leaf sibling pointers exist for range scans and iteration, not for concurrency; internal nodes omit them because top-down traversal is sufficient in a single-threaded design

