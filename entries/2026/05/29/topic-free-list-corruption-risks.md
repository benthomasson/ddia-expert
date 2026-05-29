# Topic: What happens if a crash occurs between `allocate_page` (which updates metadata) and the subsequent WAL-protected page write — can the page be leaked or double-allocated?

**Date:** 2026-05-29
**Time:** 07:54

## Crash Between `allocate_page` and WAL-Protected Write: Page Leak Analysis

**Short answer: Yes, pages can be leaked, and under specific timing, double-allocated.** This is a genuine crash-safety gap in the B-tree storage engine.

### The Core Problem

`allocate_page` (`btree.py:82–93`) modifies metadata **directly in the data file**, outside of any WAL protection:

```python
def allocate_page(self):
    root, height, total_keys, next_free, free_head = self.read_meta()
    # ... (free list path omitted) ...
    page_num = next_free
    self.write_meta(root, height, total_keys, page_num + 1, free_head)  # <-- direct write
    return page_num
```

`write_meta` calls `_write_meta` (`btree.py:40–45`), which does `self._f.flush()` but **not** `os.fsync()`. Meanwhile, the WAL's `log_write` (`btree.py:131–137`) does both `flush()` and `fsync()`. This creates two problems:

1. **The allocation is not WAL-logged** — it's a bare metadata write to the data file.
2. **The metadata write isn't even durable** — `flush()` pushes data to the OS page cache, but without `fsync()`, a power loss or kernel panic can lose it.

### Crash Scenario 1: Page Leak (Most Likely)

1. `allocate_page()` bumps `next_free` from N to N+1 and flushes metadata.
2. **Crash occurs** before the caller writes page N's content (either directly or via WAL).
3. On recovery: metadata says `next_free = N+1` (if the flush reached disk), so page N will never be allocated again. But page N was never written — it contains zeros or garbage. **Page N is leaked.**

There is no mechanism to detect this: no allocation bitmap, no page-usage cross-reference, no post-recovery consistency check. The leaked page is silently lost forever.

### Crash Scenario 2: Double-Allocation (Less Likely, More Dangerous)

1. `allocate_page()` bumps `next_free` and flushes — but `flush()` without `fsync()` means the OS may not have persisted it.
2. The WAL entry for page N's content IS written and fsynced.
3. **Crash occurs.**
4. On recovery: WAL replays page N's content correctly. But metadata on disk still says `next_free = N` (the flush was lost). Next allocation returns page N again — **overwriting the recovered data with new content.**

### The Free-List Path Has the Same Problem

When reusing pages from the free list (`btree.py:86–91`), `allocate_page` pops from the list by updating `free_head` in metadata. A crash after this metadata write but before the page is used means the page is removed from the free list but never written to — leaked and unreachable.

### Why the WAL Doesn't Help

The B-tree's embedded WAL class (`btree.py:113–160`) protects **page data writes** only. Its `recover` method (`btree.py:142–160`) replays `(page_num, page_data)` pairs back into the data file, but it has no knowledge of the metadata page's `next_free` or `free_head` fields. Recovery can restore page content but cannot detect or fix a metadata/allocation inconsistency.

### What a Fix Would Require

The metadata update and the page write need to be **atomic** — either both happen or neither does. The standard approach: log the allocation itself in the WAL (or include the updated metadata page as a WAL entry), so recovery can either roll forward the complete operation or roll it back.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:free_page` — Has the inverse problem: writes the free-list link into the page before updating metadata, risking a different kind of leak on crash
- [function] `b-tree-storage-engine/btree.py:_write_meta` — Uses `flush()` without `fsync()`, making all metadata updates non-durable — trace every caller to understand the full blast radius
- [file] `log-structured-merge-tree/lsm.py` — Compare with the LSM tree's WAL usage, which protects memtable state through replay — a more complete crash-recovery design in the same repo
- [general] `wal-commit-protocol` — The WAL `commit` method syncs the data file then truncates the WAL; explore whether the ordering guarantees are correct if a crash hits between sync and truncate
- [function] `b-tree-storage-engine/btree.py:allocate_page` — Read the three call sites (lines 346, 388, 424) to see what happens between allocation and the subsequent page write in each split/root-creation path

---

## Beliefs

- `allocate-page-not-wal-protected` — `PageManager.allocate_page` modifies metadata directly in the data file without logging the allocation to the WAL, making it vulnerable to page leaks on crash
- `write-meta-no-fsync` — `_write_meta` calls `flush()` but not `os.fsync()`, so metadata updates are not durable against power loss or kernel panic
- `wal-recover-ignores-metadata` — `WAL.recover` replays page data writes but does not validate or restore the metadata page's `next_free` or `free_head` fields
- `no-leak-detection-mechanism` — The B-tree has no allocation bitmap, page-reachability check, or post-recovery consistency scan to detect leaked or orphaned pages
- `free-page-non-atomic` — `free_page` writes the free-list link into the page before updating metadata, creating a window where a crash can orphan the page from both the tree and the free list

