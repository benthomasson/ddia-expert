# Function: _wal_write_page in b-tree-storage-engine/btree.py

**Date:** 2026-05-29
**Time:** 07:52

## `_wal_write_page` — WAL-Protected Page Write

### Purpose

`_wal_write_page` is the BTree's crash-safe page write primitive. Every mutation to a B-tree data page flows through this method rather than calling `PageManager.write_page` directly. It enforces the **write-ahead logging protocol**: the page contents are durably recorded in the WAL *before* the data file is updated. If the process crashes between the WAL write and the data file write, `WAL.recover()` replays the logged page to bring the data file back to a consistent state.

### Contract

- **Precondition**: `page_num` must refer to a valid, allocated page (either previously allocated via `pm.allocate_page()` or an existing page being updated). `data` must be a `bytes` object representing the serialized page content — it may be shorter than `page_size` but must not be empty in any meaningful use.
- **Postcondition**: After return, the WAL file contains a checksummed record of this page write (fsynced to disk), and the data file has been updated with the padded page data (flushed but **not** fsynced — that happens at `wal.commit()`).
- **Invariant**: The WAL always contains at least as much information as needed to reconstruct any in-flight writes that haven't been committed.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `page_num` | `int` | Zero-based page number in the data file. Page 0 is metadata (written via `_wal_write_meta` instead), so this is typically ≥ 1. |
| `data` | `bytes` | Serialized page content — output of `_serialize_leaf()` or `_serialize_internal()`. May be shorter than `page_size`; will be right-padded with null bytes. |

### Return Value

None. This is a side-effect-only method.

### Algorithm

1. **Pad** `data` to exactly `self.page_size` bytes using null-byte right-padding (`ljust`). This normalizes all pages to fixed size before they touch disk, ensuring the WAL record and the data file always store identically-sized pages.
2. **WAL log**: Call `self.wal.log_write(page_num, padded)`, which appends a record `[seq | page_num | data_len | data | crc32]` to the WAL file and **fsyncs** it. After this returns, the write is durable — even a crash can't lose it.
3. **Data file write**: Call `self.pm.write_page(page_num, padded)`, which seeks to `page_num * page_size` in the data file, writes, and flushes (but does **not** fsync). The data file is only fsynced later during `wal.commit()`.

The ordering is critical: WAL first, data file second. This is the "write-ahead" guarantee — the log is always ahead of the data file.

### Side Effects

- **WAL file**: Appends an entry and fsyncs. This is the expensive durable I/O.
- **Data file**: Writes the page and flushes (buffered I/O, no fsync). The `pages_written` counter on `PageManager` is incremented.
- **No in-memory state** is modified on the BTree itself.

### Error Handling

No explicit error handling. If the underlying file I/O raises (`IOError`, `OSError`), it propagates uncaught. There is no rollback mechanism — a failure between the WAL write and the data file write is exactly the scenario WAL recovery handles.

### Usage Patterns

Every B-tree mutation calls this method — never `pm.write_page` directly for data pages:

- **`_insert`** calls it to write updated/split leaf and internal pages
- **`_delete`** calls it to write modified leaf and internal pages
- **`put`** calls it to write the new root page after a root split

A typical mutation sequence looks like:

```
_wal_write_page(...)   # one or more page writes
_wal_write_meta(...)   # update metadata (root, key count, etc.)
wal.commit(pm)         # fsync data file, then truncate WAL
```

The `commit()` at the end is essential — without it, the WAL grows without bound and recovery would replay stale writes. The caller (`put`, `delete`) is responsible for calling `commit()`.

Note: metadata page 0 is written through the sibling method `_wal_write_meta` instead, which packs the metadata struct before following the same WAL-then-write pattern.

### Dependencies

- **`self.wal`** (`WAL`): Provides `log_write()` — appends checksummed records, fsyncs.
- **`self.pm`** (`PageManager`): Provides `write_page()` — positioned write to the data file, flush (no fsync).
- **`self.page_size`** (`int`): Fixed page size configured at BTree construction; used for padding.

### Assumptions Not Enforced by Types

1. **`data` is bytes, not str** — `ljust` on a string would pad with spaces, not null bytes, producing a corrupt page.
2. **`page_num` is a valid allocated page** — writing to an unallocated page number wouldn't raise an error (the file would just grow), but it would corrupt the page allocation tracking.
3. **Caller will call `wal.commit()`** — if the caller forgets, the WAL accumulates indefinitely and recovery would replay writes from prior (already-committed) transactions.
4. **`data` fits in `page_size`** — if `data` is longer than `page_size`, `ljust` is a no-op (returns `data` unchanged), but `pm.write_page` truncates to `page_size`. The WAL would store the full untruncated data while the data file stores the truncated version — a consistency violation during recovery.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:WAL.log_write` — The WAL entry format and fsync discipline that makes crash safety work
- [function] `b-tree-storage-engine/btree.py:WAL.recover` — The replay logic that uses WAL entries to restore consistency after a crash
- [function] `b-tree-storage-engine/btree.py:WAL.commit` — The commit protocol that fsyncs the data file and truncates the WAL, completing the transaction
- [function] `b-tree-storage-engine/btree.py:BTree._wal_write_meta` — The sibling method for metadata page writes, showing why page 0 has its own path
- [general] `wal-vs-shadow-paging` — Compare WAL-based crash recovery with shadow paging (copy-on-write) as used in LMDB and SQLite's legacy rollback journal

---

## Beliefs

- `wal-before-data-invariant` — Every B-tree data page write goes through `_wal_write_page`, which always writes to the WAL (with fsync) before writing to the data file (without fsync)
- `wal-commit-required` — A WAL transaction is not complete until `wal.commit()` is called, which fsyncs the data file and truncates the WAL; callers (`put`, `delete`) are responsible for this
- `page-padding-consistency` — `_wal_write_page` pads data to `page_size` before both WAL and data file writes, ensuring the WAL record and data file page are byte-identical
- `no-direct-page-writes` — The BTree class never calls `pm.write_page` directly for data pages; all writes are routed through `_wal_write_page` or `_wal_write_meta` to maintain the write-ahead invariant

