# Topic: Why `bisect_right` is the correct choice here and how it interacts with the split strategy (separator = first key of right sibling)

**Date:** 2026-05-29
**Time:** 10:37

Now I have the full picture from the `_search` and `_insert` entries. Here's the explanation:

---

# Why `bisect_right` for Internal Nodes

## The Split Strategy Creates the Constraint

When a leaf overflows during insertion (`_insert`, leaf case step 6), the split works like this:

```
Before split (5 keys, max=4):
  leaf: [A, B, C, D, E]

After split at mid=2:
  left leaf:  [A, B]        (stays in original page)
  right leaf: [C, D, E]     (goes to newly allocated page)
  separator copied up to parent: "C"  (= right_keys[0])
```

The critical detail: **the separator key "C" stays in the right leaf**. It's *copied* up to the parent, not *moved*. After the split, "C" physically lives in the right child. This is documented in `entries/2026/05/28/b-tree-storage-engine-btree-_insert.md:64`:

> The split key (`mid_key`) is `right_keys[0]` — the first key of the right page. Note: in leaf splits, the split key is **copied** up (it remains in the right leaf), unlike internal splits where it's **promoted** out.

## Why This Forces `bisect_right`

After that split, the parent internal node looks like:

```
internal node:
  keys:     [..., "C", ...]
  children: [..., left_page, right_page, ...]
                    ↓            ↓
              [A, B]        [C, D, E]
```

Now suppose we search for key `"C"`. The separator in the parent is `"C"`, and the actual data for `"C"` lives in the **right** child. We need our binary search to route us right.

- **`bisect_left(["C"], "C")` returns 0** — routes to `children[0]`, the **left** child `[A, B]`. Key not found. **Wrong.**
- **`bisect_right(["C"], "C")` returns 1** — routes to `children[1]`, the **right** child `[C, D, E]`. Key found. **Correct.**

This is exactly what the code does (`entries/2026/05/29/b-tree-storage-engine-btree-_search.md:52-54`):

```python
ikeys, children = _deserialize_internal(data)
idx = bisect_right(ikeys, key)
return self._search(children[idx], key, depth - 1)
```

## The Complementary Choice at Leaves

At the leaf level, `_search` uses `bisect_left` instead (`_search` entry, line 48):

```python
keys, values, _ = _deserialize_leaf(data)
idx = bisect_left(keys, key)
if idx < len(keys) and keys[idx] == key:
    return values[idx]
```

`bisect_left` is correct here because we want the *leftmost* position where `key` could be — if it exists, `keys[idx] == key` is our exact match. `bisect_right` would give us the position *after* all equal keys, which is wrong for an exact-match lookup.

## The General Invariant

The two choices form a coherent system:

| Level | Function | Why |
|-------|----------|-----|
| Internal | `bisect_right` | Equal-to-separator keys must go **right**, because the separator is the first key of the right child (leaf copy-up strategy) |
| Leaf | `bisect_left` | Find the exact position of the key for direct value lookup |

The same `bisect_right` choice is used in `_insert` for internal routing (step 2 of the internal case) and in `_find_leaf` for range scan setup — every traversal through internal nodes must be consistent with the split convention.

## What Would Break If You Used `bisect_left` for Internals

If you swapped to `bisect_left` at internal nodes, every key that equals a separator would be routed to the left child — where it doesn't exist. You'd get `None` for valid lookups, and inserts would place duplicates in the wrong leaf, violating the sort invariant across the leaf chain.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_insert` — See the leaf vs. internal split logic side-by-side; leaf splits copy the key up while internal splits promote it out, and this asymmetry is why `bisect_right` works uniformly for internal routing
- [function] `b-tree-storage-engine/btree.py:_find_leaf` — Range scan traversal that also uses `bisect_right` on internal nodes; confirms the invariant holds across all read paths, not just point lookups
- [function] `b-tree-storage-engine/btree.py:_delete` — Deletion must also route through internals with `bisect_right`; worth checking whether separator keys in the parent are updated when the first key of a right child is deleted
- [general] `leaf-copy-vs-internal-promote` — The dual split strategy (copy-up for leaves, promote-out for internals) is a standard B+-tree design; understanding why they differ clarifies the entire routing model
- [function] `b-tree-storage-engine/btree.py:range_scan` — Uses the leaf sibling chain (`next_sib`) that splits maintain; the correctness of this chain depends on the same split conventions that drive the `bisect_right` choice

## Beliefs

- `btree-bisect-right-matches-copy-up-split` — Internal nodes use `bisect_right` because leaf splits copy the separator into the right child as its first key; `bisect_right` routes equal keys to the right child where the data lives
- `btree-bisect-left-for-leaf-exact-match` — Leaf lookups use `bisect_left` to find the leftmost insertion point, then check `keys[idx] == key` for exact match; `bisect_right` would overshoot
- `btree-split-asymmetry-leaf-vs-internal` — Leaf splits copy the midpoint key (it appears in both parent and right child), while internal splits promote it (it appears only in the parent); this asymmetry is why a single `bisect_right` rule works for all internal routing
- `btree-bisect-left-internal-would-misroute` — Using `bisect_left` instead of `bisect_right` at internal nodes would route separator-equal keys to the left child, producing false negatives on lookup and duplicate insertions in the wrong leaf

