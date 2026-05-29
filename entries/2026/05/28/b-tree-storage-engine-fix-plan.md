# File: b-tree-storage-engine/fix-plan.md

**Date:** 2026-05-28
**Time:** 19:18

# `b-tree-storage-engine/fix-plan.md`

## Purpose

This is a **bug triage and fix plan** for the B-tree storage engine implementation in `btree.py`. It's not code — it's a design document that catalogs six correctness and durability bugs, proposes fixes with specific line references, defines an execution order based on dependency analysis, and lists test gaps. Its role is to serve as a work queue: a developer picks up the next bug in sequence, applies the described fix, and adds the corresponding test.

The document owns the **engineering reasoning** behind the fixes — not just *what* to change, but *why* the current behavior is wrong and *in what order* fixes must land.

## Key Components

### The Six Bugs

Each bug entry follows a consistent structure: **Problem** (observable failure mode), **Fix** (concrete implementation change), **Files** (line-level pointers into `btree.py`).

| Bug | Severity | Category | Core Issue |
|-----|----------|----------|------------|
| Bug 1 | High | Durability | `PageManager.write_page` calls `flush()` but not `os.fsync()` — pages may not survive a crash |
| Bug 2 | Critical | Durability | WAL truncates before data file fsync — crash window where both WAL and data are lost |
| Bug 3 | Critical | Consistency | Metadata writes (root pointer, free list head) bypass the WAL entirely |
| Bug 4 | Low | Integrity | Checksum is a naive byte sum mod 2^32 — fails to detect byte reordering or multi-byte corruption |
| Bug 5 | Medium | Resource leak | `_delete` never frees empty leaf pages — free list exists but delete doesn't use it |
| Bug 6 | Low | Fragility | `put()` reads metadata multiple times across operations that mutate it |

### Execution Order

The plan defines a **dependency-driven execution order**, not priority order:

```
Bug 4 (checksum) ──────────────────────┐
Bug 6 (metadata reads) ───────────────┐│
Bug 1 (data file fsync) ──► Bug 2 ────┤│  (WAL commit sequence)
Bug 3 (WAL metadata)   ──► Bug 2 ────┘│
Bug 5 (delete freeing) ───────────────┘
```

Bugs 1 and 3 are **prerequisites** for Bug 2 because the new commit sequence (fsync data file → truncate WAL → fsync WAL) requires both the data file fsync mechanism (Bug 1) and metadata-through-WAL logging (Bug 3) to exist first.

### Tests to Add

Four test scenarios targeting the gaps exposed by these bugs. Notably, all four test *crash recovery* or *structural integrity* — not normal-path behavior, which the existing `test_btree.py` presumably already covers.

## Patterns

**Write-Ahead Logging (WAL) protocol.** The entire document revolves around the WAL correctness invariants from DDIA Chapter 3. The fix for Bug 2 describes the canonical WAL commit sequence: ensure protected data is durable *before* discarding the log. The current code violates this by truncating the WAL before fsyncing the data file.

**Idempotent recovery.** Bug 2's fix relies on the insight that WAL replay is always safe because page writes are idempotent — writing the same page data twice produces the same result. This eliminates the need for a commit marker entirely.

**Incremental correctness.** The plan doesn't attempt a full rewrite. It fixes each bug in isolation while respecting dependencies. Bug 5 explicitly scopes out merge/redistribute (standard B-tree rebalancing) and only handles the empty-leaf case.

## Dependencies

**What it depends on:**
- `btree.py` — all line references point here. The plan assumes intimate knowledge of `PageManager`, `WAL`, `BTree._insert`, `BTree._delete`, and `BTree._write_meta`.
- DDIA Chapter 3 concepts — WAL semantics, fsync durability guarantees, B-tree page management.

**What depends on it:**
- Any developer implementing these fixes uses this as their spec.
- `test_btree.py` — the "Tests to Add" section defines new test cases.

## Flow

The document is designed to be executed top-to-bottom through the **Execution Order** section:

1. **Bug 4** (swap `_checksum` to `zlib.crc32`) — a pure function replacement with no side effects on other components.
2. **Bug 6** (consolidate metadata reads in `put()`) — a refactor that simplifies the metadata flow but doesn't change durability semantics.
3. **Bug 1** (add `os.fsync` to `PageManager.write_page`, add `sync()` method) — introduces the `sync()` primitive.
4. **Bug 3** (route `_write_meta` through WAL) — ensures metadata changes are logged.
5. **Bug 2** (rewrite `WAL.commit()` to: fsync data → truncate WAL → fsync WAL) — consumes the primitives from steps 3 and 4.
6. **Bug 5** (free empty leaves on delete) — the most complex change, independent of the durability fixes.

## Invariants

The plan implicitly defines several invariants that the fixed code must maintain:

1. **WAL-before-data**: All page writes (including metadata page 0) must be logged in the WAL before being written to the data file.
2. **Fsync-before-truncate**: The data file must be fsynced before the WAL is truncated — this is the core durability guarantee.
3. **Idempotent replay**: WAL recovery must be safe to run at any time, even if some entries were already applied to the data file.
4. **Checksum integrity**: CRC32 must detect single-bit, byte-reorder, and multi-byte corruption in WAL entries.
5. **Page lifecycle**: Allocated pages must eventually be freed when their contents are removed (no unbounded page leaks from deletes).

## Error Handling

The document doesn't describe error handling directly, but several bugs *are* error-handling failures:

- **Bug 1**: The system doesn't handle the gap between `flush()` (userspace buffer → kernel buffer) and `fsync()` (kernel buffer → disk). This is a classic durability error where the code trusts the OS to be more durable than it is.
- **Bug 2**: The crash window between WAL truncation and data file fsync is an **unhandled failure mode** — there's no recovery path if a crash occurs in that window.
- **Bug 4**: Weak checksums mean corruption detection silently fails — corrupted WAL entries are accepted as valid during recovery, propagating corruption into the data file.

## Topics to Explore

- [file] `b-tree-storage-engine/btree.py` — The implementation being fixed; read `PageManager.write_page` (lines 71-79), `WAL.commit` (lines 133-142), and `BTree._delete` (lines 442-458) to see the bugs in context
- [file] `b-tree-storage-engine/test_btree.py` — Existing test coverage; understand what's already tested to gauge the severity of the test gaps identified in the plan
- [function] `b-tree-storage-engine/btree.py:WAL.recover` — The recovery path (lines 144-172) is the most critical code in the file; understanding it is essential for evaluating whether Bug 2's fix (removing the commit marker) is safe
- [general] `fsync-flush-distinction` — The difference between `file.flush()`, `os.fsync()`, and `os.fdatasync()` is foundational to understanding why Bug 1 and Bug 2 are durability violations, not just performance issues
- [file] `write-ahead-log/wal.py` — The standalone WAL implementation in the repo likely handles the commit sequence correctly and can serve as a reference for how Bug 2's fix should look

## Beliefs

- `btree-wal-no-data-fsync` — `PageManager.write_page` calls `flush()` but never `os.fsync()`, so data file pages are not guaranteed durable on crash even when the WAL has committed
- `btree-wal-truncate-before-data-sync` — `WAL.commit()` truncates the WAL before the data file is fsynced, creating a crash window where both the log and the data may be lost
- `btree-metadata-bypasses-wal` — `BTree._write_meta` writes directly to the data file without logging through the WAL, meaning root pointer and free list updates are not crash-safe
- `btree-delete-leaks-pages` — `_delete` removes keys from leaves but never calls `free_page` on empty leaves, causing unbounded page leaks despite a free list mechanism existing in `PageManager`
- `btree-fix-plan-execution-order` — Bugs 1 and 3 must be fixed before Bug 2 because the corrected WAL commit sequence depends on `PageManager.sync()` (from Bug 1) and metadata-through-WAL logging (from Bug 3)

