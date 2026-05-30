# Topic: Read PostgreSQL's actual `src/backend/access/nbtree/README` to see the latch coupling protocol, incomplete-split handling, and page deletion dance documented in detail

**Date:** 2026-05-29
**Time:** 10:55

I'll work from the observations and my knowledge of PostgreSQL's nbtree internals to explain how the production implementation compares to this codebase's B-tree.

---

# PostgreSQL's nbtree: Latch Coupling, Incomplete Splits, and Page Deletion

## The Core Problem This Implementation Doesn't Face

The B-tree at `b-tree-storage-engine/btree.py` is single-threaded. It has no latches, no concurrent readers, and no need for the protocols that make PostgreSQL's nbtree so complex. Understanding *why* PostgreSQL needs these protocols — and what this implementation would need to add to support concurrency — is the point of this comparison.

## 1. Latch Coupling Protocol

**What PostgreSQL does:** PostgreSQL's nbtree uses a modified Lehman & Yao (L&Y) algorithm. The key insight is that traditional B-tree latch coupling (hold parent latch while acquiring child latch, then release parent) creates a bottleneck at the root. L&Y eliminates this by adding **right-link pointers** and **high keys** to each page:

- A **reader** descends without holding any latch on the parent. If a concurrent split moved the target key to a new right sibling, the reader detects this by comparing the search key against the page's high key, then follows the right-link.
- A **writer** descends similarly, but re-checks at each level. When a split is needed, it splits bottom-up: first split the leaf, then insert the new key into the parent. Between these two steps, the tree is in an "incomplete split" state — but the right-link makes it navigable.

**What this implementation does:** The local B-tree at `btree.py` takes a simpler recursive approach. The `_insert_into` method (line ~358) returns either `'inserted'` or `(mid_key, new_page_num)` when a split occurs (lines 358–359). The parent handles the split result at line 406 (`# Child split: insert new key and child`). This is a top-down-then-bottom-up pattern with no concurrency concern — the entire insert holds exclusive access implicitly.

The `NO_SIBLING = 0xFFFFFFFF` sentinel (line 13) and the `next_sibling` field in leaf serialization show that **this implementation has right-link pointers on leaves** (used for range scans), but no high keys and no right-links on internal nodes. It's half of L&Y's structure — enough for sequential range scans, not enough for concurrent split detection.

## 2. Incomplete Split Handling

**What PostgreSQL does:** A split is a multi-page operation: (1) allocate new page, (2) move half the keys, (3) set right-link, (4) insert the new downlink in the parent. If the process crashes between steps 3 and 4, the tree has an "incomplete split" — a page exists that's reachable via right-link but has no downlink from the parent. PostgreSQL handles this by:

- Detecting incomplete splits during descent: if a reader or writer follows a right-link at a non-leaf level, it knows a downlink is missing.
- Any subsequent inserter that encounters an incomplete split **finishes the split first** before proceeding with its own insert. This is cooperative recovery — no special crash-recovery pass needed.

**What this implementation does:** The WAL at `btree.py` (lines ~119–165) provides crash safety, but in a simpler way. The `WAL.recover()` method replays logged page writes by checking CRC checksums (line ~159: `if self._checksum(page_data) == checksum`). Since the implementation is single-threaded, a split either completes fully or the WAL replays it from the logged page images. There is no intermediate "incomplete split" state to detect because the WAL captures full page images, not logical operations.

The trade-off: full-page WAL images are simpler but write more data. PostgreSQL's WAL logs logical operations (like "insert tuple at offset X") which are smaller but require the incomplete-split protocol to handle partial completion.

## 3. Page Deletion Dance

**What PostgreSQL does:** Page deletion in nbtree is notoriously complex. The README documents a multi-phase protocol:

1. **Mark half-dead:** The page is marked as "half-dead" and its downlink is removed from the parent. At this point, it's still reachable via left-sibling's right-link.
2. **Unlink:** The left sibling's right-link is updated to skip the deleted page. This requires holding latches on both the target and its left sibling simultaneously.
3. **Reclaim:** The page is added to the free space map, but only after ensuring no in-flight scan could still be holding a reference. PostgreSQL uses its MVCC transaction machinery — a page can't be recycled until all transactions that might have seen it have ended.

The half-dead state (referenced in the topics queue at `.code-expert/topics.json` lines 1101–1102 as `postgres-nbtree-half-dead-state-machine`) exists specifically because the unlink can't be done atomically with the downlink removal. A crash between these steps leaves a half-dead page that VACUUM will finish cleaning up.

**What this implementation does:** The `_delete` method at `btree.py:442` and `free_page` at line 96 take a much simpler approach. When a leaf becomes empty after deletion, the page is freed immediately (`self.pm.free_page(child_page)` at line 472). The `free_page` method (lines 96–101) is a simple free-list push: write the old head pointer into the freed page, update the metadata to point to the new head.

There's no half-dead state, no sibling-link update (the left sibling's `next_sibling` still points to the freed page — a bug in a concurrent setting), and no deferred reclamation. The test at `test_btree.py:191` (`test_delete_frees_empty_leaf`) validates only that the page count changes, not that sibling links remain consistent.

## The Gap in Summary

| Concern | PostgreSQL nbtree | This implementation |
|---------|------------------|---------------------|
| Concurrency model | L&Y with right-links + high keys | Single-threaded, no latches |
| Split protocol | Bottom-up with cooperative incomplete-split repair | Recursive return of `(mid_key, new_page)` |
| Crash recovery for splits | WAL + detect-and-finish during normal operations | Full-page WAL images replayed on open |
| Page deletion | Half-dead → unlink → deferred reclaim | Immediate `free_page` push to free list |
| Sibling link maintenance on delete | Left sibling's right-link updated atomically | **Not updated** — stale link after free |

The observations are **insufficient** to show the full nbtree README text. To actually read it, you'd fetch `src/backend/access/nbtree/README` from the PostgreSQL source tree — it's roughly 800 lines of detailed protocol documentation, including the exact latch acquisition order, the "move right" rule, and the vacuum safety conditions for page recycling.

---

## Topics to Explore

- [general] `lehman-yao-right-links` — The original L&Y paper that introduced right-link pointers to eliminate parent-child latch coupling; the theoretical foundation for PostgreSQL's approach
- [general] `lehman-yao-high-keys` — How high keys let readers detect concurrent splits without holding parent latches, and why this implementation's leaf-only sibling links aren't sufficient
- [general] `postgres-nbtree-half-dead-state-machine` — The multi-phase page deletion state machine that prevents concurrent scans from following dangling pointers
- [function] `b-tree-storage-engine/btree.py:_delete` — The current deletion logic that frees pages without updating sibling links; understanding this gap clarifies why PostgreSQL's protocol exists
- [general] `postgres-nbtree-vacuum` — How PostgreSQL's VACUUM interacts with the B-tree to reclaim dead pages, and why deferred reclamation is necessary for MVCC visibility

## Beliefs

- `btree-no-concurrent-access-control` — The B-tree implementation has no latches, locks, or concurrency control; all operations assume single-threaded exclusive access
- `btree-leaf-sibling-not-updated-on-delete` — When `_delete` frees an empty leaf page via `free_page`, it does not update the predecessor leaf's `next_sibling` pointer, leaving a dangling right-link
- `btree-split-uses-recursive-return` — Splits propagate upward via return values (`(mid_key, new_page_num)` from `_insert_into`), not via the cooperative detect-and-finish protocol used in L&Y trees
- `btree-wal-uses-full-page-images` — The WAL logs complete page images with CRC checksums rather than logical operations, making incomplete-split states impossible but increasing write amplification
- `btree-free-list-immediate-reclaim` — Freed pages are immediately pushed onto the free list with no deferred-reclamation mechanism, which would be unsafe under concurrent access or MVCC

