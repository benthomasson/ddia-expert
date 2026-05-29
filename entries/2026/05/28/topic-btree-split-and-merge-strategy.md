# Topic: How the B-Tree handles node splits (tested by `test_metadata_consistency_after_split`) and whether it implements merge-on-delete or lazy tombstones

**Date:** 2026-05-28
**Time:** 19:12

I don't have access to read from the target repository directly, so I'll work with what the observations provide — and be explicit about the gaps.

---

# B-Tree Node Splits and Delete Strategy

## What We Can See (Lines 1–200 of `btree.py`)

The first 200 lines cover the **infrastructure layer** — `PageManager`, `WAL`, and the leaf serialization helpers — but **none of the split or delete logic**. Here's what we learn about the framework those operations sit on top of:

### Page Structure

- **Metadata page (page 0)** tracks `root_page`, `height`, `total_keys`, `next_free_page`, and `free_list_head` (`btree.py:16–18`). Every split or merge must update this metadata atomically via WAL.
- **Leaf pages** carry a `next_sibling` pointer (`btree.py:56–59`), forming a linked list for range scans. Splits must maintain this chain.
- **Free list** is an in-file singly-linked list — `PageManager.free_page()` (`btree.py:96–101`) pushes a page onto the list, and `allocate_page()` (`btree.py:85–94`) pops from it before falling back to appending. This means deleted pages can be reclaimed.

### WAL for Crash Safety

The `WAL` class (`btree.py:113–170`) logs page writes with CRC32 checksums before they hit the data file. `commit()` flushes the data file then truncates the WAL. `recover()` replays all valid entries on next open. This is the mechanism that makes multi-page operations (like splits) atomic.

## What's Missing — The Split Logic

The observations only retrieved lines 1–200 of a 612-line file. The split implementation — the core of how the tree grows — lives in the remaining ~400 lines. The grep for `"split"` returned **0 matches**, which is suspicious; likely the grep ran against the wrong directory or the function uses a different name (e.g., `_split_child`, `_insert_split`, or the split is inline within a `put`/`_insert` method).

**`test_metadata_consistency_after_split` was not found** in the test file either (grep returned 0 matches). The test file's first 200 lines show `test_basic` which *does* trigger splits — inserting 5 keys with `max_keys_per_page=4` forces the first split and asserts `height == 2` — but the named test doesn't exist in the portion we can see. It may be in the remaining 100 lines of `test_btree.py` (300 total).

Based on the patterns visible, the split likely:
1. Allocates a new page via `PageManager.allocate_page()`
2. Divides keys at the midpoint, writes both halves
3. Updates the `next_sibling` pointer on the new leaf to maintain the linked list
4. Pushes the median key up to the parent (or creates a new root, incrementing `height`)
5. Updates `total_keys` in the metadata page
6. All page writes go through the WAL for atomicity

## Delete Strategy — Neither Pure Merge nor Tombstones

From what's visible in the test file, this tree uses **immediate in-place deletion with page freeing**, not tombstones:

- `test_delete_and_reinsert` (`test_btree.py:138–158`) deletes keys and immediately checks `len(tree) == 10`, showing `total_keys` is decremented at delete time — no lazy tombstone compaction.
- `test_delete_frees_empty_leaf` (`test_btree.py:161+`) explicitly tests that "deleting all keys from a non-root leaf should free the page," confirming the tree reclaims empty leaves via `PageManager.free_page()`.

However, whether it does **merge-on-delete** (rebalancing underfull nodes by merging siblings or redistributing keys, like a classic B+ tree) is **not answerable from the observations**. The delete implementation is in the unread portion of `btree.py`. The test evidence suggests a simpler strategy: delete the key in-place, and if a leaf becomes completely empty, free it. This is a common pragmatic choice — full merge/redistribute is complex and often unnecessary for workloads that aren't deletion-heavy.

## Summary

| Question | Answer |
|----------|--------|
| Split mechanism | Not visible in observations (lines 201–612 unread) |
| `test_metadata_consistency_after_split` | Not found — may be in unread portion of test file or may not exist |
| Tombstones? | No — deletes are immediate, `total_keys` decrements at delete time |
| Merge-on-delete? | Unknown — delete implementation is unread, but tests suggest "free empty leaf" rather than full rebalancing |
| Crash safety during splits | WAL ensures atomicity of multi-page writes |

**To fully answer this question, we need lines 200–612 of `btree.py` (the `put`/`_insert` and `delete` methods) and lines 200–300 of `test_btree.py`.**

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:put` — The insert path contains the split logic; reading it reveals how keys are divided, how the median propagates up, and how sibling pointers are maintained
- [function] `b-tree-storage-engine/btree.py:delete` — Determines whether this is simple key removal + empty-leaf freeing, or full B-tree merge/redistribute
- [file] `b-tree-storage-engine/test_btree.py` — Lines 200–300 contain the remaining tests including `test_delete_frees_empty_leaf` completion and possibly `test_metadata_consistency_after_split`
- [general] `split-atomicity-via-wal` — Trace how a split that touches 3+ pages (old leaf, new leaf, parent, possibly new root, metadata) is made atomic through WAL entries
- [general] `free-list-reuse-under-churn` — Insert/delete cycles should show pages being reclaimed from the free list rather than the file growing unboundedly

## Beliefs

- `btree-metadata-tracks-height-and-total-keys` — The metadata page (page 0) maintains `root_page`, `height`, `total_keys`, `next_free_page`, and `free_list_head`, updated on every structural change
- `btree-delete-is-immediate-not-tombstone` — Deletes decrement `total_keys` immediately and the key becomes unreadable; no tombstone compaction pass is needed
- `btree-empty-leaf-freed-on-delete` — When all keys are removed from a non-root leaf, the page is returned to the free list via `PageManager.free_page()`
- `btree-wal-provides-split-atomicity` — Multi-page operations (splits, deletes that free pages) are made crash-safe by writing all page modifications to the WAL before applying them to the data file
- `btree-leaves-form-sibling-linked-list` — Each leaf stores a `next_sibling` page number (or `NO_SIBLING` sentinel), enabling efficient range scans without traversing internal nodes

