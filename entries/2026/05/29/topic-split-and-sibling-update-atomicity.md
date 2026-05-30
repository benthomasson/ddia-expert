# Topic: How the WAL ensures that a leaf split and its sibling pointer update are crash-atomic; a broken chain would silently truncate range scans

**Date:** 2026-05-29
**Time:** 10:41

# How the WAL Ensures Crash-Atomic Leaf Splits

## The Problem

When a leaf overflows during insertion, the B-tree must perform three coordinated writes:

1. **New right page** — contains the upper half of keys, inherits the old `next_sibling` pointer
2. **Modified left page** — retains the lower half, its `next_sibling` now points to the new right page
3. **Modified parent** — gains a separator key and child pointer to the new right page

If the process crashes after only some of these writes reach disk, the leaf sibling chain used by `range_scan` (`b-tree-storage-engine/btree.py`) could be broken. A `range_scan` walks the chain via `next_sibling` pointers — if a pointer leads to an uninitialized or corrupt page, or if the chain skips the new right page entirely, keys silently disappear from range results.

## The WAL Protocol

The crash-safety mechanism has three layers:

### 1. Write-ahead ordering via `_wal_write_page`

Every page mutation flows through `_wal_write_page` (documented in the `_wal_write_page` entry), which enforces a strict protocol:

```
WAL log entry (with fsync)  →  data file write (flush only, no fsync)
```

Each WAL entry is individually fsynced to disk before the data file is touched. This means the WAL always contains at least as much information as the data file. If the process dies at any point, the WAL has a durable record of every intended write up to that point.

### 2. Transaction grouping via `put`

The `put` method (documented in the `put` entry) structures a complete mutation as a single WAL transaction:

```
_insert(root, key, value, height)     # WAL-logs all page writes (leaf, split pages, parent updates)
  └─ _wal_write_page(right_page)      # ① new right leaf (fsync'd to WAL)
  └─ _wal_write_page(left_page)       # ② modified left leaf (fsync'd to WAL)
  └─ _wal_write_page(parent)          # ③ updated parent node (fsync'd to WAL)
_wal_write_meta(...)                   # ④ metadata (new root, key count, etc.)
wal.commit(pm)                         # ⑤ fsync data file, then truncate WAL
```

The critical property: **`_insert` never calls `wal.commit()`** (documented in the `_insert` entry as `btree-insert-no-wal-commit`). All page writes accumulate in the WAL, and only `put` calls `commit()` after everything is written. The commit sequence in `WAL.commit` (documented in the `commit` entry) is:

1. `page_manager.sync()` — fsync the data file
2. Truncate the WAL to zero
3. fsync the WAL file

This ordering ensures that the data file is durable before the WAL is cleared. If a crash occurs during commit, either the WAL still exists (and recovery replays it) or the data file is already consistent.

### 3. Idempotent replay via `WAL.recover`

On every startup, `BTree.__init__` unconditionally calls `WAL.recover()` (documented in the `recover` entry). Recovery reads the entire WAL, verifies each entry's CRC32 checksum, and replays valid entries by overwriting the corresponding pages in the data file. Then it fsyncs the data file and truncates the WAL.

The replay is **idempotent** — writing the same page image twice produces the same result. Entries with failed checksums (torn writes from a crash mid-write) are silently skipped.

## How the Sibling Chain Stays Consistent

The order of writes within `_insert` during a leaf split is significant:

1. The **right page is written first** — it contains the upper half of keys and inherits the old `next_sibling` value
2. The **left page is written second** — its `next_sibling` is updated to point to the new right page

Because each `_wal_write_page` call fsyncs the WAL entry before proceeding, there are only three possible states after a crash:

| Crash point | WAL contains | Recovery result | Sibling chain |
|---|---|---|---|
| Before right page WAL entry completes | Nothing (or torn entry, skipped by checksum) | Pre-split state | Intact — old chain unchanged |
| After right page logged, before left page | Right page only | Right page exists but left page still has old sibling pointer | Intact — left still points to old next, right page is orphaned but unreachable |
| After both logged, before commit | Both pages | Both replayed — left points to right, right points to old next | **Correct post-split chain** |

The key insight: **the sibling chain never enters a state where it points to a nonexistent or corrupt page.** The left page's sibling pointer is only updated after the right page (the target of that pointer) is already durable in the WAL.

## Known Gaps

The entries document two real gaps in the crash-atomicity story:

**1. No commit marker in the WAL** — Unlike the standalone `WriteAheadLog` in `write-ahead-log/wal.py` (which has explicit `OP_COMMIT` records and `append_batch` for atomic groups), the B-tree's `WAL` class has no transaction boundaries. `recover()` replays ALL valid entries unconditionally. This means a crash mid-split replays a **prefix** of the intended writes — not all-or-nothing. The sibling chain itself is safe (due to write ordering), but the tree structure can be inconsistent: if the parent update wasn't logged, the right page is reachable via sibling chain (range scans work) but **unreachable via tree descent** (point lookups for keys in the right page fail silently).

**2. Page allocation outside WAL coverage** — The `_insert` entry documents this as `btree-allocate-page-outside-wal`: `pm.allocate_page()` modifies metadata (the `next_free` counter or free list) through `PageManager.write_meta`, which is **not** WAL-logged. If a crash occurs after allocation but before the split completes and metadata is re-written via `_wal_write_meta`, the metadata page may be inconsistent — a page was consumed from the allocator but never populated with valid data.

## The Test Gap

The test `test_wal_recovery` in `b-tree-storage-engine/test_btree.py` (lines 107-122) validates basic crash recovery by closing file handles without calling `close()` (simulating a crash), then reopening. However, it doesn't test the mid-split crash scenario — it doesn't verify that a crash between writing the right page and the left page leaves the sibling chain intact. A more targeted test would need to inject a crash between specific `_wal_write_page` calls.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_insert` — The split logic that determines write ordering; trace the exact sequence of `_wal_write_page` calls during a leaf split and an internal split to verify the ordering guarantee
- [function] `b-tree-storage-engine/btree.py:range_scan` — The consumer of the sibling chain; understand how a broken chain manifests as silently truncated results (the scan stops at `NO_SIBLING` with no error)
- [general] `wal-commit-marker-gap` — The B-tree WAL has no transaction commit record, unlike the standalone WAL module; this means partial replays are possible, and the system relies on write ordering rather than atomicity for consistency
- [function] `b-tree-storage-engine/btree.py:allocate_page` — Page allocation modifies metadata outside WAL coverage; trace what happens if a crash follows allocation but precedes the WAL-logged metadata update
- [file] `write-ahead-log/wal.py` — Compare the standalone WAL (with `OP_COMMIT`, `append_batch`, and proper transaction boundaries) against the B-tree's simpler WAL to understand what real transaction atomicity looks like

## Beliefs

- `btree-split-write-order-protects-chain` — During a leaf split, `_insert` writes the new right page to the WAL before the modified left page, ensuring the sibling pointer target is durable before any pointer references it
- `btree-wal-no-commit-record` — The B-tree's WAL has no transaction commit marker; `recover()` replays all valid entries regardless of completeness, making crash recovery "redo-prefix" rather than "all-or-nothing"
- `btree-partial-split-replay-orphans-right-page` — If a crash occurs after the right page and left page are WAL-logged but before the parent update, recovery produces a consistent sibling chain but an unreachable right page from tree descent — range scans find the keys but point lookups do not
- `btree-allocate-page-outside-wal` — Page allocation during splits modifies metadata through `PageManager.write_meta` (not WAL-logged), creating a potential crash-safety gap if the process fails between allocation and WAL commit

