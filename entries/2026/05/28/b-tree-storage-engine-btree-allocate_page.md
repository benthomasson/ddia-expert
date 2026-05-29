# Function: allocate_page in b-tree-storage-engine/btree.py

**Date:** 2026-05-28
**Time:** 19:20

# `PageManager.allocate_page`

## Purpose

`allocate_page` is the page allocator for a file-backed B-tree storage engine. It returns the page number of a page that's safe to write to — either by recycling a previously freed page or by extending the file with a new one. This is the mechanism that prevents the data file from growing monotonically even as keys are inserted and deleted.

## Contract

**Preconditions:**
- The data file must be open and contain a valid metadata page at page 0.
- The metadata fields `next_free` and `free_head` must be consistent with the actual state of the free list and file size.

**Postconditions:**
- The returned page number is not in use by any live B-tree node.
- Metadata is updated atomically (single `write_meta` call) to reflect the allocation.
- If a free-list page was reused, `free_head` now points to the next entry in the free list.
- If no free pages existed, `next_free` is incremented by 1.

**Invariant:** After the call, the returned page number will not be returned again by a subsequent `allocate_page` until it passes through `free_page`.

## Parameters

None (beyond `self`). The allocator reads all state from the metadata page.

## Return Value

Returns an `int` — the page number that the caller may now write to. The caller is responsible for actually writing valid page data to this slot; `allocate_page` does **not** zero or initialize the page contents.

## Algorithm

1. **Read metadata** — fetches the five metadata fields: `root`, `height`, `total_keys`, `next_free`, and `free_head`.

2. **Check the free list** (`free_head != NO_SIBLING`):
   - Set the return value to `free_head` (the head of the linked list of freed pages).
   - Read the freed page's raw data to extract the pointer to the *next* free page. This pointer lives at bytes `[HEADER_SIZE : HEADER_SIZE+4]` — right after the 3-byte page header, encoded as a big-endian 4-byte unsigned int. This layout matches what `free_page` writes.
   - Update metadata with the new free-list head (`new_head`), keeping `next_free` unchanged.
   - Return the recycled page number.

3. **Append a new page** (free list is empty):
   - Set the return value to `next_free` (the first page number beyond all currently allocated pages).
   - Increment `next_free` by 1 in metadata, keeping `free_head` as `NO_SIBLING`.
   - Return the new page number.

The two paths are mutually exclusive. The free-list path is preferred when available, keeping the file compact.

## Side Effects

- **Metadata write**: Every call writes to page 0 via `write_meta`, which calls `_f.seek(0)`, `_f.write(...)`, and `_f.flush()`. This is a disk I/O operation.
- **Conditional page read**: The free-list path calls `read_page`, which increments `self.pages_read`.
- **No WAL logging**: This is a `PageManager`-level method. It writes metadata directly — it does **not** go through the WAL. The `BTree` class wraps allocation in WAL-protected operations at a higher level, but `allocate_page` itself is not crash-safe in isolation.

## Error Handling

There is no explicit error handling. The method assumes:
- The file is readable and writable (no `IOError` guard).
- The metadata page is not corrupted (no validation of the unpacked values).
- `free_head` page data is valid and contains a proper next-pointer at the expected offset.

A corrupted free-list node would silently produce a garbage `new_head` value, potentially pointing to a live page — this would cause data loss on the next allocation from the free list.

## Usage Patterns

Called by `BTree._insert` during leaf and internal node splits (to get a page for the new right sibling), and by `BTree.put` when the root itself splits (to allocate a new root page). Typical call pattern:

```python
new_page = self.pm.allocate_page()
data = _serialize_leaf(keys, values, next_sib)
self._wal_write_page(new_page, data)
```

The caller must write data to the allocated page before the next WAL commit, or the page is "leaked" — allocated in metadata but containing stale/garbage data.

## Dependencies

- **`struct`** — for unpacking the next-free pointer from the free-list node.
- **`self.read_meta()` / `self.write_meta()`** — metadata I/O on page 0.
- **`self.read_page()`** — only used in the free-list reclamation path.
- **Constants**: `NO_SIBLING` (`0xFFFFFFFF`) as the sentinel, `HEADER_SIZE` (3 bytes) to locate the free-list pointer within a freed page.

The free-list page layout is coupled to `free_page`, which writes: `HEADER_FMT` (3 bytes of zeroed header) + 4-byte pointer to the previous head. `allocate_page` reads from the same offset, so the two methods must agree on this layout or the free list breaks silently.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:free_page` — The write side of the free list; understanding the exact byte layout it writes is essential to verifying `allocate_page`'s read logic.
- [function] `b-tree-storage-engine/btree.py:_wal_write_page` — How the BTree class wraps raw page writes with WAL logging, compensating for the fact that `allocate_page` itself is not crash-safe.
- [function] `b-tree-storage-engine/btree.py:_insert` — The primary caller; shows how page allocation fits into the split-and-promote cycle of B-tree insertion.
- [general] `free-list-corruption-risks` — What happens if a crash occurs between `allocate_page` (which updates metadata) and the subsequent WAL-protected page write — can the page be leaked or double-allocated?
- [file] `b-tree-storage-engine/test_btree.py` — Verify whether tests exercise the free-list reclamation path (insert, delete, re-insert) or only the append path.

## Beliefs

- `allocate-page-prefers-free-list` — `allocate_page` always recycles from the free list before extending the file, keeping the data file compact after deletions.
- `allocate-page-not-wal-protected` — `allocate_page` writes metadata directly via `PageManager.write_meta`, not through the WAL; crash safety depends on the caller (`BTree`) wrapping the full operation in WAL commits.
- `free-list-is-singly-linked` — The free list is a singly-linked list threaded through freed pages, with each node storing a 4-byte big-endian next-pointer at offset `HEADER_SIZE` (byte 3).
- `allocate-page-no-initialization` — The returned page contains whatever data was previously on disk (stale leaf/internal node data or free-list metadata); the caller must overwrite it before the page is semantically valid.
- `free-list-layout-coupled` — `allocate_page` and `free_page` share an implicit contract on the byte layout of free-list nodes (3-byte zeroed header + 4-byte pointer); neither validates the other's output.

