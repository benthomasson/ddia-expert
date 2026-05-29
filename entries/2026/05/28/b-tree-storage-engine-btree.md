# File: b-tree-storage-engine/btree.py

**Date:** 2026-05-28
**Time:** 18:10

# `b-tree-storage-engine/btree.py`

## Purpose

This file is a self-contained, disk-backed B-tree storage engine — the kind described in DDIA Chapter 3 as the dominant indexing structure in relational databases. It owns the full stack: page-level I/O, binary serialization of leaf and internal nodes, key lookup/insert/delete with node splitting, range scans via leaf sibling pointers, and write-ahead logging for crash safety. It's a teaching implementation: correct and complete enough to demonstrate the real concepts, but not production-hardened (no concurrency, no buffer pool, no overflow pages for large values).

## Key Components

### Constants & Wire Formats

- **`INTERNAL = 0` / `LEAF = 1`** — Page type discriminators, stored as the first byte of every page.
- **`NO_SIBLING = 0xFFFFFFFF`** — Sentinel for "no next sibling" in leaf chains and "no free page" in the free list. Chosen as max uint32.
- **`META_FMT = '>5I'`** — Page 0 layout: `(root_page, height, total_keys, next_free_page, free_list_head)`. All big-endian unsigned 32-bit ints.
- **`HEADER_FMT = '>BH'`** — Every page starts with `(type: u8, num_keys: u16)` — 3 bytes.

### `PageManager`

Low-level fixed-size page I/O. Owns the data file handle and tracks read/write counts for observability.

- **`allocate_page()`** — Returns a page number, preferring the free list (a singly-linked list threaded through freed pages) before extending the file. This is important: page allocation is integrated with metadata updates, not a separate allocator.
- **`free_page(page_num)`** — Pushes a page onto the free list by writing a pointer to the old head into the freed page's body, then updating metadata.
- **`read_page` / `write_page`** — Seek-and-read/write at `page_num * page_size`. Pages are always padded to `page_size`.

### `WAL` (Write-Ahead Log)

Crash recovery via a sequential log file. Each entry is `(seq, page_num, data_len, data, crc32)`.

- **`log_write(page_num, page_data)`** — Appends an entry and fsyncs. Every page mutation goes through here before hitting the data file.
- **`commit(page_manager)`** — Fsyncs the data file, then truncates the WAL. This is the commit point: once the WAL is empty, the data file is authoritative.
- **`recover(page_manager)`** — Replays all valid (checksum-verified) entries from the WAL into the data file, then truncates. Called on startup.

### Serialization Functions

Four free functions handle the binary format of leaf and internal pages:

- **`_serialize_leaf` / `_deserialize_leaf`** — Leaf format: `header(3B) + next_sibling(4B) + entries...`. Each entry is `key_len(2B) + key + val_len(2B) + val`. Keys are UTF-8 strings; values are raw bytes.
- **`_serialize_internal` / `_deserialize_internal`** — Internal format: `header(3B) + child_0(4B) + (key_len(2B) + key + child(4B))...`. The classic B-tree layout where `n` keys separate `n+1` child pointers.

### `BTree`

The main API. Wraps `PageManager` + `WAL` and exposes a key-value interface.

**Core operations:**

- **`get(key)`** — Walks from root to leaf using `bisect_left` at leaves and `bisect_right` at internals. Returns `bytes | None`.
- **`put(key, value)`** — Recursive insert via `_insert`. Three return conventions: `None` (updated existing), `'inserted'` (new key, no split), `(mid_key, new_page)` (split happened, propagate up). When the root splits, a new root internal node is created, incrementing tree height.
- **`delete(key)`** — Recursive delete. On empty leaves, it patches the sibling pointer of the predecessor leaf and frees the empty page. Only does this cleanup when the parent is at depth 2 and the empty leaf is not the leftmost child.
- **`range_scan(start_key, end_key)`** — Finds the leaf containing `start_key`, then walks the sibling chain collecting entries until `end_key` is reached.

**Utilities:** `__contains__`, `__len__`, `__iter__`, `min_key`, `max_key`, `stats`, `print_tree`.

### `TreeStats`

A simple data class holding tree dimensions and I/O counters — useful for testing that operations touch the expected number of pages.

## Patterns

**WAL-then-write protocol.** Every page mutation follows the sequence: log to WAL (with fsync) → write to data file → eventually commit (fsync data, truncate WAL). The `_wal_write_page` and `_wal_write_meta` methods enforce this. This is the standard crash-recovery pattern from DDIA Chapter 3.

**Recursive insert with split propagation.** `_insert` returns a discriminated union (`None | 'inserted' | tuple`) that bubbles up through the call stack. Each level handles its own split locally. The root-split case is handled at the top level in `put()`, not inside the recursion — this keeps the recursion clean.

**Free list as an intrusive linked list.** Freed pages store the "next free" pointer in their own body (at offset `HEADER_SIZE`). No separate data structure needed.

**Leaf sibling chain.** Leaves store a `next_sibling` pointer enabling efficient range scans without touching internal nodes. During splits, the new right leaf inherits the old left leaf's sibling pointer, and the left leaf is updated to point to the new right leaf.

**Counter reset per operation.** `pm.reset_counters()` is called at the start of each public method, so `stats.pages_read/written` reflects the most recent operation only.

## Dependencies

**Imports:** Only stdlib — `os` (file I/O), `struct` (binary packing), `zlib` (CRC32 checksums), `bisect` (sorted-array search). Zero external dependencies.

**Imported by:** The two test files (`test_btree.py`, `tester_test_btree.py`) exercise the `BTree` API. Nothing else in the repo depends on this module.

## Flow

### Write path (`put`)

1. Reset I/O counters.
2. Read metadata (root page, height, total key count).
3. Call `_insert(root, key, value, height)` recursively:
   - At each internal level, `bisect_right` picks the child; recurse.
   - At the leaf level, `bisect_left` finds the insertion point. If key exists, overwrite. Otherwise insert.
   - If the leaf exceeds `max_keys`, split at midpoint. Return `(mid_key, new_page)` up.
   - Each internal level that receives a split result inserts the promoted key and new child pointer. If it also overflows, it splits too.
4. If the root itself split, allocate a new root with one key and two children, increment height.
5. Update metadata with new total key count.
6. Commit WAL (fsync data file, truncate WAL).

### Read path (`get`)

1. Read metadata for root and height.
2. Walk from root to leaf:
   - Internal nodes: `bisect_right` to choose child.
   - Leaf: `bisect_left` to find exact match.
3. Return the value bytes or `None`.

### Recovery path (on `BTree.__init__`)

1. Open data file and WAL file.
2. `wal.recover(pm)`: replay all valid WAL entries (checksum-verified) to the data file, fsync, truncate WAL.
3. Tree is now in a consistent state.

## Invariants

1. **Page 0 is always metadata.** The root page is never page 0.
2. **Keys within every node are sorted.** All lookup, insert, and range-scan logic depends on this.
3. **Internal nodes with `n` keys have exactly `n+1` children.** The serialization format encodes `child_0` first, then interleaves `(key, child)` pairs.
4. **Leaf sibling pointers form a left-to-right chain terminated by `NO_SIBLING`.** Splits maintain this: right inherits the old sibling, left points to right.
5. **`total_keys` in metadata is authoritative.** `__len__` reads it directly; insert/delete increment/decrement it atomically with the WAL commit.
6. **Every page write is preceded by a WAL entry.** The `_wal_write_page` / `_wal_write_meta` methods enforce this coupling.
7. **WAL commit is the atomic boundary.** If the process crashes mid-operation, recovery replays the WAL, producing a state equivalent to "all writes completed." If the WAL is empty/truncated, the data file is already consistent.

## Error Handling

- **Oversized entries:** `put()` raises `ValueError` if a single key-value pair exceeds the page size. This is the only explicit validation.
- **Missing keys:** `get()` returns `None`; `delete()` returns `False`. No exceptions for not-found.
- **Corrupt WAL entries:** `recover()` silently stops replaying at the first entry that fails the CRC32 check or is truncated. This is correct — a torn write at the tail of the WAL is expected after a crash.
- **File I/O errors:** Not caught — OS-level errors (`IOError`, `OSError`) propagate to the caller. The implementation trusts the filesystem.
- **No underflow rebalancing:** Delete can leave underfilled or empty leaf nodes (other than the one case where an empty non-leftmost leaf at depth 2 is reclaimed). This is a deliberate simplification — real B-trees merge or redistribute on underflow.

## Topics to Explore

- [file] `b-tree-storage-engine/test_btree.py` — See what invariants the tests assert: split behavior, crash recovery, range scans, and edge cases like duplicate keys
- [function] `b-tree-storage-engine/btree.py:_delete` — The delete path's limited cleanup (only depth-2 non-leftmost empty leaves) is a notable gap; understand what breaks if you remove deeper nodes
- [file] `log-structured-merge-tree/lsm.py` — The LSM-tree is the main alternative to B-trees in DDIA Chapter 3; compare the write amplification and read path tradeoffs
- [file] `write-ahead-log/wal.py` — A standalone WAL implementation in the same repo; compare its entry format and recovery semantics with the WAL embedded here
- [general] `b-tree-underflow-rebalancing` — Real B-trees (B+ trees specifically) merge or redistribute siblings on delete; this implementation skips that — explore what production databases do differently

## Beliefs

- `btree-wal-before-data` — Every page mutation is logged to the WAL (with fsync) before being written to the data file; recovery replays the WAL, so no committed write is lost on crash
- `btree-split-promotes-first-right-key` — Leaf splits promote the first key of the new right node (`right_keys[0]`) as the separator; internal splits promote the middle key and exclude it from both children
- `btree-no-underflow-merge` — Delete does not rebalance underfilled nodes; it only reclaims completely empty leaves when they are non-leftmost children of a depth-2 parent
- `btree-leaf-sibling-chain` — Leaf nodes form a singly-linked list via `next_sibling` pointers, enabling range scans without traversing internal nodes
- `btree-free-list-intrusive` — The free page list is an intrusive singly-linked list: each freed page stores the pointer to the next free page in its own body at offset `HEADER_SIZE`

