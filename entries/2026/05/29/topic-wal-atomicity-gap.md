# Topic: `allocate_page` writes metadata outside the WAL boundary; worth tracing whether this creates a crash-safety hole

**Date:** 2026-05-29
**Time:** 07:42

## `allocate_page` Writes Metadata Outside the WAL Boundary

**Short answer: yes, this is a real crash-safety hole.** The `allocate_page` and `free_page` methods in `PageManager` mutate the metadata page (page 0) directly via `write_meta`, completely bypassing the `WAL` class defined in the same file. This creates multiple crash windows where the storage engine can leak pages or corrupt the free list.

### The Vulnerable Code Path

`PageManager.allocate_page` (`b-tree-storage-engine/btree.py:84-95`) performs a read-modify-write on the metadata page:

```python
def allocate_page(self):
    root, height, total_keys, next_free, free_head = self.read_meta()
    if free_head != NO_SIBLING:
        page_num = free_head
        page_data = self.read_page(page_num)
        new_head = struct.unpack('>I', page_data[HEADER_SIZE:HEADER_SIZE+4])[0]
        self.write_meta(root, height, total_keys, next_free, new_head)  # direct write
        return page_num
    page_num = next_free
    self.write_meta(root, height, total_keys, page_num + 1, free_head)  # direct write
    return page_num
```

This calls `_write_meta` (`btree.py:37-42`), which does a raw seek-write-flush to the data file:

```python
self._f.seek(0)
self._f.write(data)
self._f.flush()      # ← flush only, no fsync
```

Compare this to `WAL.log_write` (`btree.py:130-136`), which appends to the WAL file and calls **both** `flush()` and `os.fsync()`. The metadata path has neither WAL protection nor fsync durability.

### Two Distinct Problems

**1. No fsync on metadata writes.** `_write_meta` calls `self._f.flush()` but never `os.fsync()`. The `sync()` method at line ~102 does call `os.fsync()`, but nothing in the allocation path invokes it. A crash can silently lose the metadata update even after `flush()` returns — the data may still be in the OS page cache, not on disk.

**2. Metadata mutations are not WAL-logged.** The WAL logs page data writes (B-tree node content) but never the metadata page. This means WAL recovery (`WAL.recover`, `btree.py:145-168`) replays data pages but cannot undo or redo metadata changes. The metadata and data pages can fall out of sync.

### Concrete Crash Scenarios

**Leaked page (fresh allocation):**
1. `allocate_page` advances `next_free` from 5 → 6 in metadata
2. Crash before the caller writes any data to page 5 via WAL
3. On recovery, WAL replays nothing. If metadata persisted (`next_free=6`), page 5 is permanently skipped — allocated but never used, never on the free list. It's leaked.

**Corrupted free list (reuse path):**
1. `allocate_page` reads `free_head → page 7`, follows the chain to get next pointer (`page 9`)
2. Writes metadata updating `free_head` to 9
3. Crash before page 7 is written with valid B-tree data
4. On recovery: page 7 was removed from the free list but contains stale free-list node data, not valid B-tree content. It's an orphan with garbage.

**`free_page` has the same problem** (`btree.py:97-102`): it performs two non-atomic writes (the freed page's data, then metadata) with no WAL logging and no fsync. A crash between the `write_page` and `write_meta` calls leaves the page in a state where it contains free-list data but the metadata still doesn't point to it.

### Why This Pattern Exists

The B-tree's WAL is a **redo log for page data** — it records "page N should contain these bytes" and replays that on recovery. It was not designed to handle metadata that tracks *which pages exist*. This is a common tension in storage engine design: the page allocator is logically "below" the WAL, and logging allocator state in the same WAL creates circular dependencies (you need a page to write the WAL entry about allocating a page).

Production systems solve this with techniques like:
- **Logging allocator state changes** in the WAL as a special record type (SQLite does this)
- **Deferred allocation** — reserve a page number but don't update metadata until WAL commit
- **Shadow paging** for the metadata page (write a new copy, flip a pointer atomically)
- Making the free-list updates **idempotent** so recovery can re-derive the allocator state from the page data

### Observation Gap

The grep results for WAL usage in the B-tree file returned zero matches, which is surprising since the `WAL` class is defined in `btree.py` itself. The remaining ~400 lines of `btree.py` (lines 200-612) were not included in the observations. To fully confirm the severity, we'd need to see **how `allocate_page` is called from the B-tree insert/split paths** — specifically whether those callers log the metadata page via the WAL after allocation, or whether the gap is truly end-to-end.

---

## Topics to Explore

- [file] `b-tree-storage-engine/btree.py` — Read lines 200-612 to trace how `allocate_page` is called during B-tree splits and whether any caller WAL-logs the metadata page afterward
- [function] `b-tree-storage-engine/btree.py:free_page` — Same class of bug as `allocate_page`; trace whether any B-tree delete/merge path protects this with the WAL
- [general] `wal-commit-vs-metadata-sync-ordering` — The WAL `commit` method calls `page_manager.sync()` then truncates the WAL; trace whether metadata is ever synced as part of this commit fence
- [function] `b-tree-storage-engine/btree.py:WAL.recover` — Verify whether recovery handles the case where metadata's `next_free` is ahead of the highest page actually written
- [general] `shadow-paging-or-steal-no-force` — DDIA Chapter 3 discusses alternative crash-safety strategies; compare this implementation against the textbook models

---

## Beliefs

- `allocate-page-bypasses-wal` — `PageManager.allocate_page` mutates the metadata page (page 0) via `write_meta` without logging the change in the WAL, creating a crash-safety gap between allocator state and page data
- `write-meta-no-fsync` — `PageManager._write_meta` calls `flush()` but never `os.fsync()`, so metadata updates are not guaranteed durable even when they appear to have been written
- `wal-is-redo-only-for-page-data` — The B-tree WAL (`WAL.log_write`) records page-number-to-data mappings and replays them on recovery; it has no record type for metadata or allocator state changes
- `free-page-same-vulnerability` — `PageManager.free_page` performs two non-atomic direct writes (page data + metadata) without WAL logging, exposing the same crash-safety hole as `allocate_page`

