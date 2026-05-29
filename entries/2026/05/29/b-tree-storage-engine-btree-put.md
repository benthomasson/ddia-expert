# Function: put in b-tree-storage-engine/btree.py

**Date:** 2026-05-29
**Time:** 07:37

## `BTree.put(key, value)` — Insert or Update a Key-Value Pair

### Purpose

`put` is the primary write path for the B-tree storage engine. It inserts a new key-value pair or updates an existing one, handling three distinct outcomes: in-place updates, simple insertions, and root splits that grow the tree height. All writes go through the WAL (write-ahead log) for crash safety — if the process dies mid-operation, the WAL can replay or discard the partial write on recovery.

### Contract

**Preconditions:**
- `key` must be a UTF-8 encodable string.
- `value` must be a `bytes` object (the serialization format uses `len(value)` directly with no encoding step).
- The encoded key + value + per-entry overhead must fit within a single page. This is checked explicitly.

**Postconditions:**
- After `put` returns, the key-value pair is durably stored — the WAL has been committed (truncated after syncing the data file).
- `total_keys` in metadata is incremented by 1 if a new key was inserted, unchanged if an existing key was updated.
- The tree's sorted-order invariant is maintained.

**Invariants preserved:**
- Every leaf is reachable from the root at exactly `height` levels of traversal.
- Keys within each node remain sorted.
- Leaf sibling pointers form a valid linked list for range scans.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `str` | The lookup key. Encoded to UTF-8 for storage. No length limit enforced independently — the page-size check catches oversized keys. |
| `value` | `bytes` | The payload. Stored as raw bytes. Treated as opaque by the tree. |

### Return Value

Returns `None` in all cases. This is a fire-and-forget write — success is implicit (no exception = committed). There is no way for the caller to distinguish "inserted new" from "updated existing" without a prior `get`.

### Algorithm

**Step 1 — Size validation.** Encodes the key and computes the worst-case entry size: page header (3B) + next-sibling pointer (4B) + key-length prefix (2B) + key bytes + value-length prefix (2B) + value bytes. If this exceeds `page_size`, the entry can never fit in any leaf, so it raises immediately. Note: this is a conservative lower bound — it checks whether a leaf with *only this one entry* would fit, not whether it fits alongside existing entries.

**Step 2 — Read metadata.** Fetches the root page number, tree height, total key count, next-free-page pointer, and free-list head. Resets I/O counters first for per-operation stats tracking.

**Step 3 — Recursive insert.** Calls `_insert(root, key, value, height)`, which traverses from root to the target leaf, performing the insert and handling splits bottom-up. `_insert` returns one of three values:

- **`None`** — The key already existed; its value was updated in-place. No structural change.
- **`'inserted'`** — New key was added and it fit without splitting. The leaf (and possibly internal nodes) were rewritten via WAL.
- **`(mid_key, new_page)`** — The root node itself split. A new sibling page was created, and the caller must create a new root.

**Step 4 — Handle each case:**

- **`None` (update):** Just commit the WAL. The leaf page was already written by `_insert`. Metadata doesn't change — `total_keys` stays the same.

- **`'inserted'` (no root split):** Re-read metadata (because `_insert` may have allocated pages, changing `next_free` and `free_head`), increment `total_keys`, write updated metadata through the WAL, then commit.

- **`(mid_key, new_page)` (root split):** This is the tree-growth case. Allocate a fresh page for a new root, serialize it as an internal node with one key (`mid_key`) and two children (old root + new sibling), write it through the WAL, update metadata with the new root, incremented height, and incremented key count, then commit.

**Step 5 — WAL commit.** In all three paths, `self.wal.commit(self.pm)` syncs the data file (`fsync`) then truncates the WAL to zero. This is the atomic durability boundary — once the WAL is truncated, the writes are committed.

### Side Effects

- **Disk I/O:** Reads pages along the root-to-leaf path. Writes modified leaf/internal pages and metadata to both the WAL file and the data file.
- **Page allocation:** May allocate 1+ new pages (from free list or by extending the file) if splits occur.
- **WAL state:** The WAL accumulates entries during `_insert`, then is truncated on commit. Between the first `_wal_write_page` and `commit`, the WAL contains uncommitted data that would be replayed on crash recovery.
- **Counter reset:** `pm.reset_counters()` zeroes the read/write page counters, affecting `stats` if called concurrently.

### Error Handling

- **`ValueError`** — Raised if the entry is too large for the page size. This is the only explicit error. The operation has no side effects at this point (raised before any I/O).
- **Implicit failures:** Disk I/O errors (`OSError`, `IOError`) from `write_page`, `fsync`, or WAL operations are not caught — they propagate to the caller. If an exception occurs after WAL writes but before commit, the WAL retains the partial writes. On next startup, `WAL.recover()` replays them, which is safe because WAL entries are idempotent page overwrites.
- **No validation** that `value` is actually `bytes`. Passing a `str` would fail inside `_serialize_leaf` at `struct.pack('>H', len(v)) + v` when concatenating with bytes.

### Usage Patterns

```python
tree = BTree('/tmp/mydb')
tree.put('user:1001', b'{"name": "Alice"}')
tree.put('user:1001', b'{"name": "Alice", "active": true}')  # update
tree.close()
```

Callers are expected to:
- Encode values to bytes before calling (the tree stores opaque bytes).
- Call `close()` when done to flush and release file handles.
- Not call `put` concurrently — there is no locking. The re-read of metadata after `_insert` (to pick up changed `next_free`/`free_head`) assumes single-threaded access.

### Dependencies

- **`PageManager`** — All page I/O (read, write, allocate). The `put` method relies on `allocate_page()` updating metadata as a side effect, which is why it re-reads metadata after `_insert`.
- **`WAL`** — `log_write` for journaling, `commit` for atomic durability. The WAL's `recover` method on startup replays any incomplete `put`.
- **`_serialize_internal`, `_serialize_leaf`** — Page serialization used when creating the new root on split.
- **`_insert`** — The recursive core that does the actual tree traversal, insertion, and split logic.
- **`struct`** — Binary packing for the size validation calculation.

### Notable Assumptions

1. **Single-writer assumption.** The metadata re-read pattern (`_read_meta()` after `_insert`) assumes no concurrent writer has modified it. There are no locks or CAS operations.
2. **`value` is bytes.** Not enforced by a type check — passing a string silently produces incorrect behavior or a crash in serialization.
3. **Page size is sufficient for metadata + at least one entry.** The size check validates the entry alone, but doesn't account for the case where `max_keys` might be computed to a value that can't actually hold that many entries of this size.
4. **`allocate_page` mutates metadata as a side effect.** This is why `put` re-reads metadata after `_insert` returns — the `next_free` and `free_head` fields may have changed during splits.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_insert` — The recursive insertion engine that handles leaf updates, splits at every level, and propagates split results upward
- [function] `b-tree-storage-engine/btree.py:WAL.commit` — The atomic durability boundary — understanding when data transitions from "recoverable" to "committed"
- [function] `b-tree-storage-engine/btree.py:WAL.recover` — How crash recovery replays WAL entries to restore consistency after a partial `put`
- [file] `b-tree-storage-engine/test_btree.py` — Test cases revealing edge cases: maximum-size entries, sequential vs random insert patterns, crash-during-split scenarios
- [general] `btree-concurrency-control` — This implementation has no locking — explore how production B-trees (e.g., InnoDB, PostgreSQL) use latching and lock coupling for concurrent access

---

## Beliefs

- `btree-put-wal-atomicity` — Every `put` operation writes all modified pages to the WAL before writing to the data file, and commits by syncing the data file then truncating the WAL — crash at any point is recoverable
- `btree-put-three-outcomes` — `_insert` returns exactly one of three values: `None` (update, no key count change), `'inserted'` (new key, no root split), or `(mid_key, new_page)` (root split requiring a new root node)
- `btree-single-writer` — `put` assumes single-threaded access with no concurrent writers; it re-reads metadata after `_insert` without any concurrency control
- `btree-root-split-grows-height` — When the root splits, `put` creates a new root with height+1, making the tree grow uniformly from the top — all leaves remain at the same depth
- `btree-put-metadata-reread` — `put` must re-read metadata after `_insert` returns because page allocation during splits mutates `next_free` and `free_head` as a side effect of `PageManager.allocate_page`

