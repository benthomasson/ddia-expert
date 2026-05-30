# Topic: How a leaf split propagates upward when the parent is also full, and how root splits increase tree height

**Date:** 2026-05-29
**Time:** 10:45

Now I have the full picture. Here's the explanation:

---

# How Leaf Splits Propagate Upward and Root Splits Increase Tree Height

## The Core Mechanism: Recursive Return Values

The entire split propagation system hinges on a single design choice in `_insert`: its **three-way return value** acts as a message from child to parent about what happened.

| Return value | Meaning | Parent's obligation |
|---|---|---|
| `None` | Key existed, value updated in-place | Nothing |
| `'inserted'` | New key added, node had room | Nothing structural |
| `(mid_key, new_page)` | **Node split** | Insert `mid_key` as a separator, add `new_page` as a new child |

When `_insert` recurses downward from root to leaf (decrementing `depth` at each level), the split signal propagates back *upward* through the call stack. Each level either absorbs the split (if it has capacity) or splits again and passes the signal further up.

## Step by Step: A Cascading Split

Suppose we insert a key into a tree where both the target leaf **and** its parent internal node are full (`len(keys) > max_keys` after insertion).

### Phase 1: Leaf Split (`depth == 1`)

1. `_insert` reads and deserializes the leaf page into `keys`, `values`, `next_sib`.
2. Uses `bisect_left` to find the insertion point; inserts the new key-value pair.
3. The leaf now has `max_keys + 1` entries — overflow.
4. **Split at the midpoint**: `mid = len(keys) // 2`.
   - Left page (original): keeps `keys[:mid]`, `values[:mid]`.
   - Right page (newly allocated via `pm.allocate_page()`): gets `keys[mid:]`, `values[mid:]`.
5. **Sibling chain maintenance**: the right page inherits the old `next_sib` pointer; the left page's `next_sib` is set to the new right page number. This preserves the leaf linked list for range scans.
6. `mid_key = right_keys[0]` — the split key is **copied** (it stays in the right leaf).
7. Returns `(mid_key, new_page)` to the caller.

### Phase 2: Parent Internal Node Absorbs... or Splits (`depth > 1`)

The parent's `_insert` call receives `(mid_key, new_page)` from the recursive call.

1. Inserts `mid_key` at position `idx` in `ikeys`.
2. Inserts `new_page` at position `idx + 1` in `children` (immediately after the child that split).
3. Now checks: does this internal node overflow?

**If the parent has room** (`len(ikeys) <= max_keys`): serialize the updated internal node, write it via WAL, return `'inserted'`. The split stops here.

**If the parent is also full**: the internal node splits:
   - Split at `mid = len(ikeys) // 2`.
   - The key at `ikeys[mid]` is **promoted** — it does *not* appear in either child. This differs from leaf splits where the key is copied.
   - Left: `ikeys[:mid]` with `children[:mid + 1]`.
   - Right: `ikeys[mid + 1:]` with `children[mid + 1:]`.
   - Returns `(promote_key, new_page)` to *its* caller.

This cascading pattern continues up the tree. Each level either absorbs the split or propagates it further.

### Phase 3: Root Split — the Tree Grows Taller

When the split signal reaches `put` (the top-level caller), it means **the root itself split**. This is the only situation where the tree's height increases.

`put` handles it by creating an entirely new root:

1. Allocates a fresh page via `pm.allocate_page()`.
2. Serializes it as an internal node with:
   - One key: `mid_key` (the promoted separator)
   - Two children: the old root page (left) and `new_page` (right)
3. Writes the new root through the WAL.
4. Updates metadata: `root_page = new_root`, `height += 1`, `total_keys += 1`.
5. Commits the WAL (fsync data file, then truncate WAL).

This is why **B-trees grow from the top, not the bottom** — all leaves remain at the same depth, and a new level is added above the old root. The tree tests confirm this: inserting 5 keys with `max_keys_per_page=4` forces the first split and asserts `height == 2` (`test_btree.py:21`).

## Leaf vs. Internal Split: A Critical Difference

| Aspect | Leaf split | Internal split |
|---|---|---|
| Split key | **Copied** — remains in the right leaf | **Promoted** — removed from both children |
| Sibling pointer | Right inherits old `next_sib`; left points to right | No sibling chain for internal nodes |
| Why the difference | Leaf keys must be findable via tree descent; the split key lives in the right leaf | Internal keys are separators only; promoting avoids duplication |

## Crash Safety During Cascading Splits

Every page write within `_insert` goes through `_wal_write_page`, which logs the page image to the WAL with an fsync **before** writing to the data file. `_insert` never calls `wal.commit()` — only `put` does, after all pages (including the new root, if created) and metadata are written. This means the entire cascading split is a single WAL transaction: either all page writes are replayed on recovery, or none are (if the WAL entries have invalid checksums from a torn write).

One known gap: `pm.allocate_page()` modifies metadata outside WAL coverage. If a crash occurs after allocation but before the WAL-logged metadata update in `put`, the allocator's state could be inconsistent.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_insert` — The recursive engine handling both leaf and internal splits; trace the exact `bisect_right` routing and how `depth` controls leaf vs. internal behavior
- [function] `b-tree-storage-engine/btree.py:_delete` — Deletion uses a simpler strategy (no rebalancing, just free empty leaves); worth contrasting with the split-heavy insert path
- [general] `wal-atomicity-gap` — `allocate_page` modifies metadata outside WAL coverage during splits; a crash between allocation and commit could leave phantom pages
- [function] `b-tree-storage-engine/btree.py:range_scan` — The consumer of the leaf sibling chain that `_insert` maintains during splits; a broken chain silently truncates scan results
- [general] `btree-concurrency-control` — This implementation is single-writer with no latching; production B-trees (InnoDB, PostgreSQL) use latch coupling to allow concurrent splits

## Beliefs

- `btree-splits-propagate-via-return-value` — `_insert` communicates splits to the parent through a `(mid_key, new_page)` return tuple; each ancestor either absorbs the split or returns its own tuple, cascading up to `put`
- `btree-root-split-grows-height` — When the root splits, `put` creates a new root with `height + 1`, making the tree grow uniformly from the top so all leaves remain at the same depth
- `btree-leaf-copies-internal-promotes` — Leaf splits **copy** the midpoint key (it stays in the right leaf for lookups), while internal splits **promote** it (removed from both children, exists only in the parent)
- `btree-insert-no-wal-commit` — `_insert` never commits the WAL; the caller `put` is solely responsible for calling `wal.commit` after all page writes complete, making the entire cascading split a single atomic WAL transaction
- `btree-sibling-chain-maintained` — During leaf splits, the right page inherits the old `next_sib` and the left page's `next_sib` points to the new right page, preserving the linked list for range scans

