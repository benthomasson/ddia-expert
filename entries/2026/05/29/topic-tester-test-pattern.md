# Topic: The `tester_test_*.py` convention appears in ~25 project directories; understanding its role vs. `test_*.py` clarifies the project's testing strategy

**Date:** 2026-05-29
**Time:** 07:36

# The `tester_test_*.py` vs `test_*.py` Convention

This project uses a **two-tier testing strategy**: each implementation directory contains both a `test_*.py` file and a `tester_test_*.py` file. They look similar at first glance — both use pytest conventions, both test the same module — but they serve different roles in the development pipeline.

## `tester_test_*.py`: The Spec Verification Suite

These files are **generated as part of the implementation task**. They're written by the "tester" stage of the code-expert workflow — an LLM agent that reads the implementation spec and produces tests that verify the spec's requirements are met.

Key characteristics:

- **Spec-aligned test names**: Tests mirror spec language. In `consistent-hashing/tester_test_consistent_hashing.py:7`, `test_determinism` has a docstring `"""Same key always maps to same node."""` — this reads like a spec requirement, not a developer's unit test.
- **Docstrings on every test**: Compare `tester_test_btree.py:8` (`"""Test basic insert and lookup from the example usage."""`) with `test_btree.py:8` (no docstring, just `def test_basic`). The tester files consistently document *what requirement* each test validates.
- **Print-based pass reporting**: Every test in `tester_test_btree.py` ends with `print("test_xxx PASSED")` (lines 46, 65, 83, etc.) and includes a `__main__` block (line 192) that runs all tests sequentially. This suggests they were designed to run standalone, outside pytest, as a quick validation pass.
- **Conservative, focused coverage**: `tester_test_btree.py` has 8 tests covering the core operations. `test_btree.py` has 10+ tests and goes deeper — WAL recovery (line 119), uncommitted WAL entries (line 144), delete freeing empty leaves (line 171).

## `test_*.py`: The Developer Test Suite

These are the **comprehensive tests** that go beyond spec verification:

- **Implementation-aware tests**: `test_btree.py:test_wal_recovery` (line 119) simulates a crash by closing file handles directly (`tree.pm._f.close()`) — this tests internal recovery mechanics, not a user-facing spec requirement.
- **Internal state manipulation**: `test_btree.py:test_wal_uncommitted_entries` (line 144) imports internal types (`from btree import WAL, _serialize_leaf, HEADER_FMT, LEAF`) and manually writes WAL entries. The tester file never touches internals.
- **Additional edge cases**: `test_consistent_hashing.py` includes `test_duplicate_add_idempotent` (line 108), `test_ring_position_valid_range` (line 114), `test_scalability` (line 120), and `test_vnode_count_affects_balance` (line 153) — none of which appear in the tester version.

## The Naming Variants

Not every project follows the exact `tester_test_*.py` pattern. `leaderless-replication/test_dynamo_tester.py` inverts the convention to `test_*_tester.py`, and `unbundled-database/test_tester_validation.py` uses yet another variant. The docstring in `test_dynamo_tester.py:1` — `"""QA tester tests for Dynamo-style leaderless replication."""` — confirms the "tester" label refers to the QA/validation role regardless of filename ordering. The `unbundled-database/test_tester_validation.py` file (line 1: `"""Tester-stage validation: edge cases and spec example verification."""`) makes this even more explicit: these are "tester-stage" artifacts with classes like `TestSpecExample` that replay the exact usage from the spec.

## Why Two Suites?

The split reflects the code-expert workflow documented in `CLAUDE.md`: implementations are built by an LLM pipeline that generates code from DDIA-based specs. The tester files are a **correctness gate** — "does this implementation satisfy what the spec asked for?" The developer test files are the **robustness layer** — "does this implementation handle crashes, edge cases, and internal invariants correctly?" Running `pytest` picks up both (all files match `test_*.py` or `*_test_*.py`), so the full suite runs together, but the tester files could also run standalone via `python tester_test_*.py`.

## Topics to Explore

- [general] `tester-generation-pipeline` — How the code-expert workflow generates `tester_test_*.py` files from specs, and whether they're regenerated when specs change
- [file] `unbundled-database/test_tester_validation.py` — The most explicit tester file: class names (`TestSpecExample`, `TestEdgeCases`) reveal the tester stage's internal structure
- [file] `leaderless-replication/test_dynamo_tester.py` — Uses the inverted naming convention; worth checking if the naming inconsistency reflects an evolution in the project's conventions
- [general] `test-coverage-gaps` — Whether tester files ever test things the developer suite doesn't, or if the developer suite is always a strict superset
- [file] `b-tree-storage-engine/test_btree.py` — Lines 119-183 show the depth gap: WAL crash recovery, internal state manipulation, and page-freeing tests that the tester file never touches

## Beliefs

- `tester-files-are-spec-verification` — `tester_test_*.py` files verify spec requirements with docstrings and print-based reporting; `test_*.py` files test implementation internals and edge cases
- `tester-files-never-import-internals` — Tester test files import only the public API (e.g., `from btree import BTree`), while developer test files import internal types like `WAL`, `_serialize_leaf`, and `HEADER_FMT`
- `tester-files-run-standalone` — Every `tester_test_*.py` includes a `__main__` block that calls each test function sequentially, independent of pytest
- `naming-convention-not-uniform` — The tester/test split uses at least three naming patterns: `tester_test_*.py`, `test_*_tester.py`, and `test_tester_*.py` across different project directories
- `both-suites-discoverable-by-pytest` — Both naming conventions match pytest's default collection patterns, so `pytest` in any project directory runs the full combined suite

