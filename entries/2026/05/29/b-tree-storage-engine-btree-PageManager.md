# Function: PageManager in b-tree-storage-engine/btree.py

**Date:** 2026-05-29
**Time:** 11:02

## `PageManager` — Fixed-Size Page I/O Layer

### Purpose

`PageManager` is the lowest-level storage abstraction in this B-tree implementation. It treats a single file as an array of fixed-size pages (default 4096 bytes — one OS page / one disk sector on most systems) and provides read/write access by page number. It also manages page allocation and a free list so that deleted pages can be reclaimed without compacting the file.

This exists to separate the concerns of *how bytes get to disk* from *what the B-tree structure looks like*. The `BTree` class works entirely in terms of page numbers and serialized byte buffers; `PageManager` handles the file seeks, padding, and space management.

### Contract

**Preconditions:**
- `file_path` must be writable. If the file doesn't exist, the parent directory must exist.
- `page_size` must be large enough to hold the metadata format (`META_FMT` = 20 bytes) and at least one leaf entry.
- Page numbers passed to `read_page`/`write_page` must be within the allocated range (0 through `next_free - 1`), or you'll read zeros / silently extend the file.

**Postconditions:**
- After `__init__`, page 0 always contains valid metadata and page 1 is always a valid (possibly empty) leaf root.
- Every `write_page` and `_write_meta` call flushes the userspace buffer (`flush()`), but does **not** call `fsync` — data may still be in the OS page cache. Only `sync()` and `close()` guarantee durability.
- `allocate_page` always returns a valid, writable page number and updates metadata atomically (from the caller's perspective).

**Invariants:**
- Page 0 is always the metadata page. User pages start at page 1.
- The free list is a singly-linked list threaded through freed pages: each freed page stores the next free pointer at offset `HEADER_SIZE` (byte 3). The head of the list is in metadata field `free_head`. `NO_SIBLING` (0xFFFFFFFF) terminates the list.
- `next_free` is a high-water mark — the next page number that has never been used. It only increases.
- `pages_read` and `pages_written` track I/O counts since the last `reset_counters()` call, used by `BTree.stats`.

### Parameters

**`__init__(self, file_path, page_size=4096)`**

| Parameter | Type | Description |
|-----------|------|-------------|
| `file_path` | `str` | Path to the data file. Created if absent. |
| `page_size` | `int` | Bytes per page. Must be consistent across the file's lifetime — there is no on-disk record of page size, so reopening with a different value silently corrupts reads. |

### Return Values

- `read_meta()` → `tuple[int, int, int, int, int]`: `(root_page, height, total_keys, next_free_page, free_list_head)`. All big-endian unsigned 32-bit integers.
- `read_page(page_num)` → `bytes`: Always exactly `page_size` bytes, zero-padded if the file was short.
- `allocate_page()` → `int`: The page number of the newly allocated page. Caller is responsible for writing meaningful content to it.
- `free_page(page_num)` → `None`: The freed page is prepended to the free list.

### Algorithm

**Initialization** (`__init__`):
1. Open the file in `r+b` (existing) or `w+b` (new).
2. If new: write metadata at page 0 with `root=1, height=1, total_keys=0, next_free=2, free_head=NO_SIBLING`. Then write an empty leaf at page 1. The tree starts as a single empty leaf.

**Page allocation** (`allocate_page`):
1. Read current metadata.
2. If `free_head != NO_SIBLING` — a previously freed page exists:
   - Read that page's data.
   - Extract the "next" pointer from `data[HEADER_SIZE:HEADER_SIZE+4]` (the second link in the free list).
   - Update metadata to point `free_head` to that next pointer.
   - Return the recycled page number.
3. Otherwise, bump `next_free` by 1 and return the old value. This extends the file on the next write.

**Page freeing** (`free_page`):
1. Read current metadata to get the current `free_head`.
2. Overwrite the freed page with a minimal header (type=0, num_keys=0) followed by a 4-byte pointer to the old `free_head`. This makes the freed page a node in the free list.
3. Update metadata so `free_head` points to this newly freed page.

This is a classic **intrusive singly-linked free list** — the free list nodes are stored *inside* the freed pages themselves, requiring no additional bookkeeping space.

### Side Effects

- **File I/O**: Every `read_page` seeks and reads. Every `write_page` and `write_meta` seeks, writes, and flushes.
- **I/O counters**: `pages_read` and `pages_written` are mutated on every page access. Note that `read_meta` does **not** increment `pages_read`, so metadata reads are invisible to the stats.
- **File growth**: `allocate_page` can cause the file to grow when `next_free` advances past the current file size — the actual growth happens on the subsequent `write_page`.
- **No locking**: There is no file lock, `fcntl`, or thread synchronization. Concurrent access from multiple processes or threads is unsafe.

### Error Handling

- If the file can't be opened, Python's `open()` raises `OSError`/`FileNotFoundError` — not caught here.
- `read_page` on a page beyond the file's current size returns a short read, which is zero-padded to `page_size`. This silently succeeds rather than raising — a page number typo won't crash, it'll return garbage.
- `write_page` silently truncates data longer than `page_size` (`data[:self.page_size]`). Oversized writes are clipped without warning.
- No `try/finally` on the file handle — if `__init__` fails after `open()` but before completing setup, the handle leaks. In practice this can only happen if `_write_meta` or `_write_empty_leaf` raises (e.g., disk full).

### Usage Patterns

`PageManager` is always used through `BTree`, never directly by application code:

```python
# BTree.__init__ creates and owns the PageManager
self.pm = PageManager(data_path, page_size)

# Typical lifecycle inside BTree:
data = self.pm.read_page(page_num)       # read
self.pm.write_page(page_num, new_data)   # write
new_page = self.pm.allocate_page()       # grow
self.pm.free_page(old_page)              # reclaim
self.pm.sync()                           # durability barrier
self.pm.close()                          # shutdown
```

Caller obligations:
- Never write to page 0 directly — use `write_meta`.
- Call `sync()` or `close()` when durability matters. The WAL layer handles this in practice.
- Call `reset_counters()` before each operation if you want per-operation I/O stats.

### Dependencies

- `os` — file existence check, `fsync`
- `struct` — binary serialization using `META_FMT` (">5I" = 5 big-endian unsigned 32-bit ints) and `HEADER_FMT` (">BH" = 1 unsigned byte + 1 unsigned 16-bit short)
- Module-level constants: `META_FMT`, `META_SIZE`, `HEADER_FMT`, `HEADER_SIZE`, `LEAF`, `NO_SIBLING`

No external dependencies. Pure stdlib.

### Assumptions Not Enforced

1. **Page size consistency**: Nothing on disk records what `page_size` was used. Reopening with a different value silently produces corrupt reads.
2. **Single-writer**: No file locking. Two processes opening the same file will corrupt it.
3. **`flush()` ≠ durability**: Most writes call `flush()` but not `fsync()`. On crash, recently written pages may be lost. The `BTree` class compensates via the WAL, but `PageManager` alone does not guarantee durability.
4. **Page number validity**: `read_page` and `write_page` accept any `int` — there's no bounds check against `next_free`. Reading an unallocated page returns zeros; writing one silently extends the file.
5. **Free list integrity**: If the file is corrupted such that a free-list pointer forms a cycle, `allocate_page` will loop forever, reusing the same pages.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:BTree._wal_write_page` — How the WAL wraps PageManager writes to provide crash safety that PageManager alone lacks
- [general] `intrusive-free-list-pattern` — Why storing free-list pointers inside freed pages is space-efficient and how it compares to bitmap-based allocation
- [function] `b-tree-storage-engine/btree.py:BTree.put` — The primary consumer of `allocate_page`, showing how page splits drive allocation
- [general] `fsync-ordering-guarantees` — Why `flush()` without `fsync()` is insufficient for durability and what failure modes this creates
- [general] `btree-concurrency-control` — What it would take to make PageManager safe for concurrent access (file locks, latches, copy-on-write)

---

## Beliefs

- `page-manager-flush-not-durable` — PageManager calls `flush()` on every write but only calls `fsync()` in `sync()` and `close()`, so individual writes are not crash-durable without the WAL layer.
- `page-manager-free-list-is-intrusive` — Freed pages store the next-free pointer at offset HEADER_SIZE inside the page data itself, forming a singly-linked list with no external bookkeeping.
- `page-manager-no-page-size-on-disk` — The page size is not persisted in the data file; reopening with a different page_size silently corrupts all reads.
- `page-manager-allocate-prefers-free-list` — `allocate_page` recycles from the free list before bumping the high-water mark, keeping file size stable under delete-heavy workloads.
- `page-manager-meta-reads-untracked` — `read_meta()` does not increment `pages_read`, so metadata access is invisible to `TreeStats.pages_read`.

