# Function: _write_meta in b-tree-storage-engine/btree.py

**Date:** 2026-05-29
**Time:** 10:58

## `_write_meta` — PageManager metadata page writer

### Purpose

`_write_meta` serializes the B-tree's global metadata into page 0 of the data file. Page 0 is reserved as the "superblock" — it holds the five integers that define the tree's current state. Every structural mutation (insert, delete, split, page allocation, page free) eventually calls this to persist the updated tree shape.

### Contract

**Preconditions:**
- `self._f` is an open, writable file handle.
- All five arguments are unsigned 32-bit integers (0 to 2³²−1). This is not enforced — `struct.pack('>5I', ...)` will raise `struct.error` if a value is negative or exceeds 4 bytes.

**Postconditions:**
- Page 0 contains exactly `self.page_size` bytes: the 20-byte packed metadata followed by zero-padding.
- The data has been flushed to the OS buffer cache (but **not** fsynced to disk — a crash could lose it).

**Invariant:** Page 0 is always `page_size` bytes and always starts with a valid `META_FMT` header.

### Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `root` | `int` | Page number of the current root node. Starts at 1 (page 0 is metadata). |
| `height` | `int` | Tree height: 1 means the root is a leaf, 2+ means internal nodes exist above leaves. |
| `total_keys` | `int` | Total key-value pairs stored across all leaves. Maintained manually by callers. |
| `next_free` | `int` | Next page number to allocate when the free list is empty. Monotonically increases. |
| `free_head` | `int` | Head of the intrusive free list (a singly-linked list of deallocated pages). `NO_SIBLING` (0xFFFFFFFF) means the list is empty. |

### Return Value

None. This is a pure side-effect method.

### Algorithm

1. **Pack** the five integers into 20 bytes using big-endian unsigned format (`'>5I'`).
2. **Pad** to `page_size` with null bytes via `ljust`. This ensures page 0 occupies exactly one page — the same size as every data page — so page arithmetic (`page_num * page_size`) remains valid.
3. **Seek to offset 0** — page 0 always lives at the start of the file.
4. **Write** the padded buffer.
5. **Flush** the Python-level buffer to the OS. This calls `fflush` semantics, pushing data out of Python's `io.BufferedWriter` into the kernel's page cache.

### Side Effects

- **File I/O**: overwrites the first `page_size` bytes of the data file.
- **File position**: leaves the file cursor at byte `page_size` (just past page 0).
- **No fsync**: `flush()` does not guarantee durability. A power failure after `_write_meta` but before `os.fsync` can lose the metadata update. The `BTree` class compensates by routing through `_wal_write_meta`, which logs to the WAL with `fsync` before writing the data file. However, `PageManager.__init__` calls `_write_meta` directly during initialization — an unprotected write.

### Error Handling

No explicit error handling. Possible exceptions:
- `struct.error` if any argument isn't a valid unsigned 32-bit int.
- `OSError` / `IOError` if the file write fails (disk full, closed handle).

Both propagate uncaught to the caller.

### Usage Patterns

Two distinct call sites with different safety profiles:

1. **`PageManager.__init__`** — called directly during first-time file creation. No WAL protection exists yet; a crash here leaves a partially-initialized file. Acceptable because there's no data to lose.

2. **`PageManager.write_meta` / `BTree._write_meta`** — the public wrapper delegates here. The `BTree` class's `_wal_write_meta` logs to the WAL *before* calling this, providing crash recovery. `allocate_page` and `free_page` call `write_meta` directly (not through WAL), which is a durability gap — though these are always followed by WAL-protected writes in the `put`/`delete` paths.

### Dependencies

- `struct` — binary packing via `META_FMT` (`'>5I'`, 20 bytes big-endian).
- `self._f` — a raw file handle opened in `__init__`.
- `self.page_size` — determines the padding width; must be ≥ 20 (META_SIZE).

### Assumptions not enforced by types

- `page_size >= META_SIZE` (20 bytes). If `page_size < 20`, the `ljust` is a no-op but the packed data still overwrites into page 1's territory.
- Arguments represent a consistent tree state. Nothing prevents writing `root=999` when only 3 pages exist — the corruption surfaces later on read.
- The file handle is open and writable. No check for closed handles.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_wal_write_meta` — The WAL-protected counterpart; compare the durability guarantees with the unprotected `_write_meta`
- [function] `b-tree-storage-engine/btree.py:allocate_page` — Consumes `next_free` and `free_head` from metadata; shows how the free list interacts with page allocation
- [general] `directory-fsync-gap` — Whether `flush()` without `fsync` creates a real durability window on common filesystems
- [function] `b-tree-storage-engine/btree.py:_serialize_leaf` — How leaf page data is laid out; contrast the fixed-layout metadata page with variable-length leaf entries
- [general] `intrusive-free-list-pattern` — How deallocated pages form a linked list embedded in the page data itself

---

## Beliefs

- `meta-page-is-page-zero` — The B-tree metadata (root, height, total_keys, next_free, free_head) is always stored at file offset 0, occupying exactly one page
- `meta-flush-without-fsync` — `_write_meta` calls `flush()` but not `os.fsync()`, so metadata writes are not durable against power loss unless the caller uses the WAL path
- `meta-page-fixed-20-byte-payload` — The metadata payload is always exactly 20 bytes (5 × 4-byte big-endian unsigned ints), zero-padded to `page_size`
- `init-meta-unprotected-by-wal` — During initial file creation, `PageManager.__init__` writes metadata and the root leaf page without WAL protection — acceptable because no user data exists yet
- `allocate-free-bypass-wal` — `allocate_page` and `free_page` call `write_meta` directly rather than through `_wal_write_meta`, relying on the caller's subsequent WAL commit for atomicity

