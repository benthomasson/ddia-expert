# File: b-tree-storage-engine/test_btree.py

**Date:** 2026-05-28
**Time:** 18:28

# `b-tree-storage-engine/test_btree.py`

## Purpose

This is the test suite for a disk-backed B-Tree storage engine — one of the core data structure implementations from *Designing Data-Intensive Applications*. It validates the B-Tree as a complete key-value store with on-disk persistence, write-ahead logging, and crash recovery. The file owns correctness verification across the full surface area: CRUD operations, range queries, page-level I/O behavior, durability guarantees, and data integrity (CRC32 checksums).

## Key Components

The suite is 11 standalone test functions, no class hierarchy — each creates a fresh `BTree` in a `tempfile.TemporaryDirectory` so tests are fully isolated.

### Core functionality tests

- **`test_basic`** — The integration smoke test. Exercises put, get, `__contains__`, range scan, iteration order, min/max key, update-in-place, delete, stats, and **persistence across close/reopen**. This single test covers more surface area than most of the others combined.
- **`test_large`** — Inserts 1000 keys with zero-padded names (`key_00000`–`key_00999`) and verifies all retrievals, sorted iteration, and that the tree has grown beyond height 1. Confirms the B-Tree scales past trivial sizes.
- **`test_range_scan`** — Tests bounded (`k05` to `k10`, exclusive end) and unbounded (`k15` to end) range scans. Validates that range boundaries are half-open: start-inclusive, end-exclusive.
- **`test_delete_and_reinsert`** — Deletes every even-indexed key, reinserts them with new values, and verifies both the reinserted and untouched keys. Confirms the tree handles tombstone-to-live transitions correctly.

### Durability and crash recovery tests

- **`test_wal_recovery`** — Simulates a crash by closing raw file handles (`tree.pm._f`, `tree.wal._f`) without calling `tree.close()`. Reopens and verifies all 10 keys survived. This tests that the WAL replays uncommitted page writes on startup.
- **`test_wal_uncommitted_entries`** — Goes further: after a clean close, manually appends a WAL entry (via `WAL.log_write`) *without* a commit marker, then reopens. Verifies the tree still loads correctly — uncommitted WAL entries are replayed but shouldn't corrupt committed state.
- **`test_crc32_detects_corruption`** — Writes a WAL entry, then flips a byte inside it to corrupt the CRC. On reopen, the corrupted entry must be silently skipped while previously committed data remains intact.

### Structural integrity tests

- **`test_page_io_counts`** — After inserting 100 keys, asserts that a single `get` reads at most `height + 1` pages. Validates that lookups follow the B-Tree's O(log n) page-read guarantee, not scanning.
- **`test_too_large_kv`** — Confirms that inserting a value larger than the page can hold raises `ValueError`. The tree uses `page_size=128` and a 200-byte value to trigger this.
- **`test_delete_frees_empty_leaf`** — Deletes keys from a non-root leaf and verifies the tree stays consistent: sorted iteration works, range scans skip deleted keys, and the gap doesn't break traversal.
- **`test_metadata_consistency_after_split`** — After 20 inserts (forcing multiple splits), reads the metadata page directly (`tree._read_meta()`) and asserts the root pointer references a valid `INTERNAL` node. Reopens and verifies metadata is identical — the root pointer, height, and key count survive persistence.

## Patterns

**Temp-directory isolation.** Every test uses `with tempfile.TemporaryDirectory() as tmpdir` — the B-Tree writes real files to disk, so this gives each test a clean filesystem namespace with automatic cleanup.

**Close-reopen persistence checks.** Several tests (`test_basic`, `test_wal_recovery`, `test_metadata_consistency_after_split`) close the tree and reopen it from the same directory to verify durability. This is the signature pattern for testing storage engines.

**Internal-API probing.** Tests like `test_page_io_counts` reach into `tree.pm.pages_read`, and `test_metadata_consistency_after_split` calls `tree._read_meta()` and `_page_type()`. These aren't black-box tests — they verify internal invariants that a pure API test can't catch.

**Lazy imports.** Several tests import B-Tree internals (`WAL`, `_serialize_leaf`, `_page_type`, `INTERNAL`, `HEADER_FMT`, `LEAF`) inside the function body rather than at module scope. This keeps the top-level import surface minimal and makes it clear which tests depend on internals.

**`max_keys_per_page=4`.** Nearly every test uses this small fanout to force splits with few keys, making tree height and structure predictable for assertions.

## Dependencies

**Imports:**
- `tempfile`, `os`, `struct` — standard library, for test scaffolding and low-level WAL manipulation
- `btree.BTree` — the system under test
- `btree.WAL`, `btree._serialize_leaf`, `btree.HEADER_FMT`, `btree.LEAF`, `btree._page_type`, `btree.INTERNAL` — internal APIs used only in durability/corruption tests

**Imported by:** Nothing — this is a leaf test module. The companion file `tester_test_btree.py` likely wraps or re-runs these tests in a harness.

## Flow

Each test follows the same arc:

1. Create a temp directory
2. Instantiate `BTree(tmpdir, max_keys_per_page=4)` — this either creates fresh files or opens existing ones
3. Perform writes (`put`), reads (`get`), deletes (`delete`), or scans (`range_scan`, iteration)
4. Assert correctness of results, stats, or internal state
5. Optionally close and reopen to test persistence
6. `tree.close()` (or simulate crash by skipping it)
7. Print `"test_X PASSED"` — the tests use print-based reporting, not a framework like pytest

When run as `__main__`, all 11 tests execute sequentially, and a final "All tests passed!" confirms the full suite.

## Invariants

- **Sorted iteration**: every test that iterates the tree asserts `all_keys == sorted(all_keys)` — the B-Tree must maintain key order.
- **Range scan boundaries**: `range_scan(start, end)` is start-inclusive, end-exclusive — `test_range_scan` asserts `"k05"` is included but `"k10"` is not.
- **Page read bound**: a single lookup must touch at most `height + 1` pages (one per tree level plus possibly the data page).
- **Value size limit**: keys/values that exceed `page_size` must raise `ValueError`, never silently corrupt.
- **WAL crash safety**: data committed before a crash must survive; uncommitted WAL entries must not corrupt existing data.
- **CRC integrity**: corrupted WAL entries are skipped, not applied.
- **Metadata consistency**: `_read_meta()` must return accurate root page, height, and key count, and these must survive close/reopen.

## Error Handling

The tests themselves don't handle errors — they assert expected outcomes and let `AssertionError` propagate on failure. The one exception is `test_too_large_kv`, which uses a try/except to verify that `BTree.put` raises `ValueError` for oversized values. WAL corruption (`test_crc32_detects_corruption`) verifies that the *implementation* handles errors silently — a corrupted entry is skipped, not propagated to the caller.

## Topics to Explore

- [file] `b-tree-storage-engine/btree.py` — The implementation being tested: page manager, WAL, split logic, and the serialization format (`_serialize_leaf`, `HEADER_FMT`)
- [function] `b-tree-storage-engine/btree.py:BTree.range_scan` — How half-open range iteration is implemented across leaf pages
- [function] `b-tree-storage-engine/btree.py:WAL.log_write` — WAL entry format, CRC32 checksumming, and the commit protocol that `test_wal_uncommitted_entries` probes
- [file] `b-tree-storage-engine/tester_test_btree.py` — The test harness wrapper; understanding how it relates to this file clarifies the project's test infrastructure
- [general] `btree-split-and-merge-strategy` — How the B-Tree handles node splits (tested by `test_metadata_consistency_after_split`) and whether it implements merge-on-delete or lazy tombstones

## Beliefs

- `btree-range-scan-half-open` — `BTree.range_scan(start, end)` uses half-open intervals: start-inclusive, end-exclusive; passing no end scans to the last key
- `btree-max-keys-forces-splits` — With `max_keys_per_page=4`, inserting 5+ keys forces at least one split, producing a tree of height >= 2
- `btree-wal-replays-without-commit` — The WAL replays all valid entries on recovery regardless of whether a commit marker is present
- `btree-crc32-skips-corrupt-entries` — Corrupted WAL entries (bad CRC) are silently skipped during recovery rather than raising an error
- `btree-lookup-reads-bounded-by-height` — A single `BTree.get` reads at most `height + 1` disk pages

