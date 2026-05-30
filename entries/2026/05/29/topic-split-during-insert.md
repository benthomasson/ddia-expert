# Topic: Examine how leaf splits maintain the sibling chain invariant (the new leaf must be linked in correctly or range scans break)

**Date:** 2026-05-29
**Time:** 10:54

# How Leaf Splits Maintain the Sibling Chain Invariant

## The Invariant

Every leaf in this B+-tree carries a 4-byte `next_sibling` pointer (stored immediately after the 3-byte page header), forming a singly-linked list across all leaves in sorted key order. The rightmost leaf uses the sentinel `NO_SIBLING = 0xFFFFFFFF` (`btree.py:13`). If this chain is ever broken — a leaf inserted without being linked, or a pointer left dangling — `range_scan` and `__iter__` silently lose data.

## The Split Rewiring (in `_insert`, leaf case)

When a leaf overflows during insertion (key count exceeds `max_keys`), the split must thread the new page into the existing chain. The algorithm, documented in the `_insert` entry and visible from the code's structure:

1. **Split the keys at the midpoint**: `mid = len(keys) // 2`. Left half stays in the original page; right half goes to a newly allocated page.

2. **Right page inherits the old `next_sib`**: The original leaf's existing sibling pointer passes to the new right page. This is critical — the right page takes the original leaf's position in the chain relative to everything to the right.

3. **Left page points to the new right page**: The original leaf's `next_sib` is rewritten to the new page number.

The two `_serialize_leaf` calls during a split look conceptually like:

```python
# Right half gets the old forward pointer
right_data = _serialize_leaf(right_keys, right_values, next_sibling=old_next_sib)

# Left half now points to the new right page
left_data = _serialize_leaf(left_keys, left_values, next_sibling=new_page_num)
```

This is a **linked-list insert-after** operation: the new node splices in between the original node and its former successor.

## Why This Ordering Matters

Consider a chain `[A] → [B] → [C]` where leaf B overflows and splits into B (left) and B' (right):

```
Before:  [A] → [B] → [C]
After:   [A] → [B] → [B'] → [C]
```

- B' inherits B's old pointer to C (`next_sib = C`)
- B's pointer is updated to B' (`next_sib = B'`)
- A's pointer doesn't change — it already points to B

If you got this backwards — if B' pointed to B instead of C, or B still pointed to C instead of B' — a `range_scan` would either loop or skip the new right half entirely.

## Where the Chain Is Consumed

**`range_scan`** (`btree.py`): Descends to the start key's leaf via `_find_leaf`, then walks `next_sibling` pointers horizontally. Cost is O(height + leaf pages in range). A broken link truncates results silently — there's no cycle detection (`range_scan` entry, line 63).

**`__iter__`** (`btree.py`): Descends the left spine (always following `children[0]`) to reach the leftmost leaf, then walks the entire chain to `NO_SIBLING`. A broken link in the middle means `list(tree)` silently drops everything after the break.

Both rely on the chain being **complete** (every leaf reachable) and **acyclic** (terminates at `NO_SIBLING`). Neither validates these properties at runtime.

## Deletion Also Maintains the Chain

`_delete` handles the mirror operation: when an empty leaf is removed (`btree.py`, `_delete` entry), it patches the **previous** sibling's `next_sibling` to point to the **removed leaf's** `next_sibling`, splicing the empty node out. This only happens at depth 2 for non-leftmost children — an empty leftmost leaf stays allocated, wasting a page but not breaking the chain.

## The Crash-Safety Gap

The split writes two pages (left and right halves) via `_wal_write_page`, which logs to the WAL before writing to the data file. If the process crashes after writing the right page but before updating the left page's `next_sibling`, the chain would be broken: the new right page exists but nothing points to it. WAL replay should fix this (it replays all logged writes), but `allocate_page` modifies metadata through `PageManager.write_meta` **outside** the WAL boundary — so there's a window where the page is allocated but the chain update hasn't been logged.

## The Wire Format That Makes It Work

From `_serialize_leaf` (`btree.py:182-191`):

```
[page_type: 1B] [num_keys: 2B] [next_sibling: 4B] [entries...]
```

The `next_sibling` field is at a **fixed offset** (bytes 3-6), so `_deserialize_leaf` can always find it regardless of how many entries the page holds. This is what makes the chain walkable without parsing the entire page.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_find_leaf` — The descent logic that locates a starting leaf for range scans; uses `bisect_right` routing at internal nodes
- [function] `b-tree-storage-engine/btree.py:put` — The public entry point that calls `_insert`, handles root splits (creating a new root with two children), and commits the WAL
- [general] `split-atomicity-via-wal` — Trace the exact sequence of WAL writes during a split to verify whether a crash between allocating the new page and updating sibling pointers can orphan data
- [function] `b-tree-storage-engine/btree.py:_delete` — The asymmetric counterpart: deletion only cleans up empty leaves at depth 2, and `free_page` bypasses the WAL entirely
- [general] `concurrent-iteration-safety` — Neither `range_scan` nor `__iter__` guards against concurrent `put`/`delete`; a split during iteration can corrupt the cursor's view of the chain

---

## Beliefs

- `btree-split-inherits-sibling` — During a leaf split, the new right page inherits the original leaf's `next_sibling` pointer, and the original leaf's `next_sibling` is rewritten to point to the new right page — a linked-list insert-after operation
- `btree-sibling-chain-fixed-offset` — The `next_sibling` field sits at a fixed byte offset (3-6) in every leaf page, making the chain walkable without fully parsing page contents
- `btree-chain-no-cycle-detection` — Neither `range_scan` nor `__iter__` detects cycles in the sibling chain; a corrupted `next_sibling` pointer causes an infinite loop
- `btree-delete-patches-prev-sibling` — When `_delete` removes an empty leaf, it reads the previous sibling and rewrites its `next_sibling` to point past the removed leaf, maintaining chain continuity
- `btree-split-wal-gap` — Page allocation during splits modifies metadata outside the WAL, creating a window where a crash could leave the new page allocated but not linked into the sibling chain

