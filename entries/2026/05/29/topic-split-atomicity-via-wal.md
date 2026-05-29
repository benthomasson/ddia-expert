# Topic: Trace how a split that touches 3+ pages (old leaf, new leaf, parent, possibly new root, metadata) is made atomic through WAL entries

**Date:** 2026-05-29
**Time:** 07:40

The file is outside the current project's directory. Let me work with what's in the observations, which include the first 200 lines. The critical split logic is beyond line 200 and wasn't captured, but the WAL class and PageManager are fully visible. Let me provide a thorough analysis of what we *can* see and what's missing.

# How B-Tree Splits Are Made Atomic Through WAL Entries

## What the Code Shows

### The WAL Class (`b-tree-storage-engine/btree.py`, lines ~120–170)

The B-tree's embedded `WAL` class implements a **physical page-level WAL** — each WAL entry stores a complete page image:

```
Entry format: seq(4B) + page_num(4B) + data_len(4B) + page_data + checksum(4B)
```

Three methods form the core protocol:

1. **`log_write(page_num, page_data)`** (line ~137): Appends a full page image to the WAL file, then `fsync`s. Each call increments a sequence counter, so entries are ordered. The `fsync` after every entry guarantees durability before the caller proceeds.

2. **`commit(page_manager)`** (line ~144): Calls `page_manager.sync()` to flush all dirty data pages to disk, then **truncates the WAL to zero**. This is the atomicity boundary — once the WAL is truncated, the transaction is committed.

3. **`recover(page_manager)`** (line ~150): On startup, replays all valid WAL entries by writing each page image back to the data file, using checksum validation to skip torn entries. After replay, it truncates the WAL.

### The PageManager (`b-tree-storage-engine/btree.py`, lines ~27–115)

`PageManager` handles raw page I/O. Key detail: `write_page` (line ~80) calls `flush()` but **not** `fsync()` — individual page writes are not durable on their own. Only `sync()` (line ~108) calls `os.fsync()`. This is deliberate: the WAL is the durability mechanism, not individual page writes.

## How a Multi-Page Split Would Work

A leaf split that promotes a key to the parent (and possibly creates a new root) touches at minimum:

| Page | What changes |
|------|-------------|
| Old leaf (page N) | Keys truncated to first half |
| New leaf (page M) | Gets second half of keys |
| Parent internal node | Gets new separator key + pointer to page M |
| Metadata (page 0) | `next_free_page` incremented, possibly `root_page` and `height` updated |
| New root (if needed) | Created with two children pointers |

The protocol for atomicity would be:

1. **Log phase**: Call `wal.log_write()` for each modified page — old leaf, new leaf, parent, metadata, (new root). Each `log_write` is individually `fsync`'d, so the WAL accumulates a durable record of all intended changes.

2. **Apply phase**: Call `page_manager.write_page()` for each page. These writes are buffered (no `fsync`).

3. **Commit phase**: Call `wal.commit()`, which `fsync`s the data file (making all page writes durable) then truncates the WAL.

### Crash Scenarios

| Crash point | State | Recovery |
|-------------|-------|----------|
| During log phase (partial WAL entries) | Data file untouched | `recover()` replays only entries with valid checksums. Partial entries at the end are skipped — the split never happened. The tree is consistent at its pre-split state. |
| After all WAL entries written, during apply phase | WAL is complete, data file partially updated | `recover()` replays **all** WAL entries, re-writing every page. The split completes. |
| After apply, before WAL truncation | Both WAL and data file are complete | `recover()` replays entries idempotently (writing the same page images again). No harm done. |
| After WAL truncation | Committed | No recovery needed. |

### The Key Insight: Redo-Only WAL

This is a **redo-only** (no undo) WAL. It stores the *after-images* of pages. The protocol guarantees:

- The WAL is always written and `fsync`'d **before** the data file is modified
- Recovery always replays forward — it re-applies the intended page writes
- Checksum validation (`zlib.crc32`, line ~170) ensures torn WAL entries are detected and ignored
- The WAL truncation at commit is the linearization point — it's the single atomic action that transitions the system from "uncommitted" to "committed"

## What's Missing from the Observations

**The actual split and insert code was not captured.** The observations include only lines 1–200 of a 612-line file. The split logic (likely in an `_insert` or `_split_leaf` method) and the B-tree's `put()` method that orchestrates WAL logging around splits are beyond line 200. Without that code, I cannot confirm:

- The exact order of `log_write` calls during a split
- Whether the code logs all pages before writing any (strict WAL protocol) or interleaves logging and writing
- How root splits are handled — whether a new root is logged before or after the metadata update
- Whether there's any transaction/batch grouping beyond the implicit "everything between the last commit and the next commit"

The `write-ahead-log/wal.py` file is a **separate, standalone WAL** implementation (operation-level, not page-level) that isn't used by the B-tree. The B-tree has its own embedded `WAL` class.

## Topics to Explore

- [file] `b-tree-storage-engine/btree.py` — Read lines 200–612 to see the actual split logic, insert path, and how `wal.log_write()` calls are sequenced around multi-page mutations
- [function] `b-tree-storage-engine/btree.py:_serialize_leaf` — Understanding the page format is essential to reasoning about what recovery replays
- [general] `redo-vs-undo-logging` — This WAL uses redo-only logging (after-images); contrast with undo logging (before-images) and ARIES-style redo/undo to understand the tradeoffs
- [general] `fsync-ordering-guarantees` — The correctness of this WAL depends on `fsync` ordering between the WAL file and data file; explore what guarantees the OS and filesystem actually provide
- [file] `write-ahead-log/wal.py` — The standalone WAL implementation uses operation-level records with COMMIT markers and batching — a different design point worth comparing to the B-tree's page-level WAL

## Beliefs

- `btree-wal-is-redo-only` — The B-tree WAL stores full after-images of pages (not deltas or before-images), making recovery a simple forward replay with no undo capability
- `btree-wal-fsync-per-entry` — Each `WAL.log_write()` call performs an `fsync`, guaranteeing that WAL entries are durable before any data page writes proceed
- `btree-page-writes-not-individually-durable` — `PageManager.write_page()` calls `flush()` but not `fsync()`; data pages are only durable after `PageManager.sync()` is called during WAL commit
- `btree-wal-truncation-is-commit-point` — The WAL truncation in `commit()` is the atomic linearization point: before truncation, recovery will redo the transaction; after truncation, the transaction is committed
- `btree-has-separate-wal-from-standalone` — The B-tree's embedded `WAL` class (page-level, in `btree.py`) is entirely separate from the standalone `WriteAheadLog` class (operation-level, in `wal.py`)

