# File: b-tree-storage-engine/tester_test_btree.py

**Date:** 2026-05-28
**Time:** 19:11

# `b-tree-storage-engine/tester_test_btree.py`

## Purpose

This is the **standalone test harness** for the B-Tree storage engine implementation. It validates the `BTree` class against the behavioral contract described in DDIA Chapter 3 — a disk-backed, page-oriented sorted index supporting insert, lookup, update, delete, range scans, and crash recovery.

The naming convention `tester_test_*.py` appears throughout the repo (every project directory has one). These are the "tester" variants — self-contained test scripts that run via `python tester_test_btree.py` without pytest, printing `PASSED`/`FAILED` to stdout. The sibling `test_btree.py` likely uses pytest conventions. This dual-harness pattern lets the project validate implementations both in CI (pytest) and interactively (direct execution).

## Key Components

Eight test functions, each targeting a distinct property of the B-Tree:

| Function | What it validates |
|---|---|
| `test_basic_put_get` | CRUD lifecycle: insert, lookup, update, delete, membership (`in`), min/max, `len()` |
| `test_range_scan` | Bounded range (`k05..k10`), unbounded range (`k18..end`), full sorted iteration |
| `test_persistence` | Close-and-reopen preserves data, updates, and deletes (crash recovery) |
| `test_large_dataset` | 1000-key correctness — all lookups match, iteration is sorted, tree is balanced |
| `test_page_io_counts` | Lookup cost is O(height) — reads at most `height + 1` pages per `get()` |
| `test_value_too_large` | Rejects key-value pairs that exceed page size with `ValueError` |
| `test_empty_tree` | All operations are safe on an empty tree (no crashes, sane defaults) |
| `test_delete_all_keys` | Deleting every key returns the tree to an empty-but-usable state |

## Patterns

**Temp-directory isolation.** Every test uses `tempfile.TemporaryDirectory()` as a context manager, giving the B-Tree a fresh directory for its page files. This means tests are fully independent — no shared state, no cleanup logic, safe to run in any order.

**Small branching factor.** All tests use `max_keys_per_page=4`, which forces splits early. With a realistic branching factor (hundreds of keys per page), you'd need millions of entries to exercise multi-level trees. With 4, a 5th insert triggers the first split, making structural behavior (height growth, page splits, rebalancing) observable at small scale.

**Print-based reporting.** Each function prints its own `PASSED` message. The `__main__` block runs all tests sequentially and prints a final summary. No test framework dependency — if any assertion fails, the script crashes with a traceback at that point.

**Contract-style assertions.** Tests assert on the public API contract (`get`, `put`, `delete`, `range_scan`, `min_key`, `max_key`, `len`, `__iter__`, `__contains__`) and on observable internals (`tree.stats.height`, `tree.pm.pages_read`). The stats/page-manager exposure is deliberate — it lets tests verify performance properties, not just correctness.

## Dependencies

**Imports:**
- `tempfile` — stdlib, for isolated test directories
- `btree.BTree` — the implementation under test

**Imported by:** Nothing. This is a leaf node — a test runner entry point.

**Implicit dependency on `BTree` API surface:**
- `BTree(path, max_keys_per_page=, page_size=)` constructor
- `.put(key, value)`, `.get(key)`, `.delete(key)` — CRUD
- `.range_scan(start, end=None)` — returns list of `(key, value)` tuples
- `.min_key()`, `.max_key()` — boundary accessors
- `.close()` — flush and release resources
- `.stats` — object with `.height`, `.total_pages`
- `.pm` — page manager with `.pages_read`, `.reset_counters()`
- `__len__`, `__iter__`, `__contains__` — dunder protocol support

## Flow

When run as `python tester_test_btree.py`:

1. Each test function is called sequentially in definition order.
2. Each creates a fresh `BTree` in a temp directory, exercises the API, asserts invariants, calls `.close()`, and prints `PASSED`.
3. If any assertion fails, Python raises `AssertionError` with a diagnostic message and the script halts — no further tests run.
4. If all eight pass, the final `"All tests PASSED"` line prints.

There's no setup/teardown framework — the `with tempfile.TemporaryDirectory()` block handles both. The `test_persistence` function is the only one that exercises the close-reopen cycle by creating a second `BTree` instance on the same directory.

## Invariants

- **Sorted iteration**: `test_range_scan` and `test_large_dataset` both assert `all_keys == sorted(all_keys)` — the B-Tree must maintain key ordering across all leaves.
- **No duplicates on update**: `test_basic_put_get` inserts "apple" twice and asserts `len(tree) == 10` — `put` on an existing key is an update, not an insert.
- **Height grows on split**: After the 5th insert with `max_keys_per_page=4`, height must be 2.
- **O(height) lookups**: `test_page_io_counts` asserts `pages_read <= height + 1` after a single `get()`.
- **Balance at scale**: With 1000 keys and `max_keys=4`, height must be >= 2 (in practice it will be ~5-6 for a balanced tree with branching factor 5).
- **Empty-tree safety**: All operations on an empty tree return `None`, `0`, `False`, or `[]` — never raise.
- **Reusable after full deletion**: After deleting all keys, the tree accepts new inserts normally.

## Error Handling

Minimal, by design:

- **`test_value_too_large`** is the only test that validates error production — it expects `BTree.put()` to raise `ValueError` when a key-value pair won't fit in a single page. It uses a bare `try/except ValueError` with a manual `assert False` fallthrough instead of pytest's `raises()`.
- All other tests rely on assertion failures to signal problems. The assertion messages (e.g., `f"height={tree.stats.height}"`, `f"got {keys}"`) include diagnostic context to aid debugging.
- No tests validate concurrent access, corruption recovery, or partial-write scenarios — the scope is single-threaded correctness.

## Topics to Explore

- [file] `b-tree-storage-engine/btree.py` — The implementation under test; understanding the page split algorithm, page manager, and stats tracking will contextualize every assertion in this file
- [function] `b-tree-storage-engine/btree.py:range_scan` — How leaf-level sibling pointers enable efficient range queries across page boundaries
- [file] `b-tree-storage-engine/test_btree.py` — The pytest-based sibling test suite; compare coverage and see if it tests additional edge cases (concurrent access, corruption)
- [general] `tester-test-pattern` — The `tester_test_*.py` convention appears in ~25 project directories; understanding its role vs. `test_*.py` clarifies the project's testing strategy
- [file] `b-tree-storage-engine/fix-plan.md` — Suggests there were known issues with this implementation; reading it reveals what broke and how it was fixed

## Beliefs

- `btree-max-keys-forces-splits` — Setting `max_keys_per_page=4` causes the first page split after the 5th insert, raising tree height to 2
- `btree-put-is-upsert` — `BTree.put()` on an existing key updates the value in-place without increasing `len(tree)`
- `btree-get-reads-at-most-height-plus-one-pages` — A single `get()` reads at most `height + 1` pages from disk, verified via `pm.pages_read`
- `btree-range-scan-excludes-end` — `range_scan("k05", "k10")` returns keys k05 through k09 — the end bound is exclusive
- `btree-survives-close-reopen` — Data written before `close()` is fully recoverable by constructing a new `BTree` on the same directory

