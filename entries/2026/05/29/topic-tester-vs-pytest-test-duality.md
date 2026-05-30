# Topic: Compare a `tester_test_*.py` and its corresponding `test_*.py` side-by-side to understand what each convention gains and loses (framework independence vs. fixture reuse, human output vs. structured reporting)

**Date:** 2026-05-29
**Time:** 11:29

# `tester_test_*.py` vs `test_*.py`: Two Testing Conventions

This codebase maintains two parallel test files for many modules. They look similar at first glance — both use `assert`, both test the same code — but they serve fundamentally different purposes and make different tradeoffs.

## The Core Distinction

**`tester_test_*.py`** files are self-contained scripts. They run with `python tester_test_btree.py` — no test framework needed. Every file has an `if __name__ == '__main__':` block (`tester_test_btree.py:190`) that calls each test function in sequence and prints human-readable output like `"test_basic_put_get PASSED"` (`tester_test_btree.py:45`).

**`test_*.py`** files are pytest-native. They rely on pytest discovery (function names starting with `test_`), use `pytest.raises` for exception testing, and in some modules use `@pytest.fixture` for shared setup (`test_cdc.py:10`, `test_bitcask.py:10`). They produce no output themselves — pytest's runner handles reporting.

## Side-by-Side: B-Tree Tests

| Dimension | `tester_test_btree.py` | `test_btree.py` |
|-----------|----------------------|----------------|
| **Lines** | 200 | 300+ |
| **Test count** | 8 | 10+ |
| **Dependencies** | `tempfile`, `btree` only | `tempfile`, `os`, `struct`, `btree`, WAL internals |
| **Output** | `print("test_X PASSED")` per test | None (pytest handles it) |
| **Runner** | `python tester_test_btree.py` | `pytest test_btree.py` |
| **Error testing** | Manual try/except (`line 143-149`) | `pytest.raises` |
| **Granularity** | One fat `test_basic_put_get` covers CRUD+split+range | Separate `test_basic`, `test_range_scan`, `test_delete_and_reinsert` |
| **Internal access** | None — black-box only | Imports WAL internals, manipulates file handles directly (`test_wal_recovery`, `test_wal_uncommitted_entries`) |

The tester version's `test_basic_put_get` (lines 8–45) is a single 37-line function that exercises put, get, contains, split, update, delete, min/max, and close. The pytest version splits this across `test_basic` (lines 7–56), `test_range_scan` (lines 79–93), and `test_delete_and_reinsert` (lines 140–161) — each independently runnable and independently reportable.

## What Each Convention Gains

### `tester_test_*.py` gains:

- **Zero dependencies.** No pip install needed. Run it on any Python 3 installation. This matters for educational code — a reader clones the repo and runs `python tester_test_btree.py` without knowing what pytest is.
- **Human-readable output.** Each test prints its own result, including diagnostic values: `"test_large_dataset PASSED (height=5, pages=342)"` (`tester_test_btree.py:117`). You see what happened, not just pass/fail.
- **Sequential narrative.** The `__main__` block reads like a script: basic ops → ranges → persistence → scale → IO costs → errors → edge cases. A new reader follows the implementation's capability arc top-to-bottom.
- **Framework independence.** The tester convention appears across 13 modules (`grep_tester_main` results). No coupling to pytest's lifecycle, fixture injection, or plugin system.

### `test_*.py` gains:

- **Fixture reuse.** Modules like `test_cdc.py` and `test_bitcask.py` use `@pytest.fixture` to share setup across tests — create a temp directory once, pass it to multiple test functions. The tester files repeat `with tempfile.TemporaryDirectory() as tmpdir:` in every single test.
- **Granular failure isolation.** When `test_delete_and_reinsert` fails, you know exactly what broke. In `tester_test_btree.py`, if the script dies at line 30 inside the 37-line `test_basic_put_get`, you have to read the traceback to figure out which operation failed.
- **Structured reporting.** pytest gives you `-v` for verbose, `-x` for fail-fast, `-k` for pattern matching, `--tb=short` for compact tracebacks, JUnit XML for CI. The tester files give you `print`.
- **Deeper testing.** `test_btree.py` has tests the tester doesn't: WAL recovery (`test_wal_recovery`, line 119), WAL uncommitted entries (`test_wal_uncommitted_entries`, line 152), and delete-frees-empty-leaf (`test_delete_frees_empty_leaf`, line 179). These require importing internals and simulating crashes — work that belongs in a proper test suite, not a demo script.

## What Each Convention Loses

The **tester** convention can't do parameterized tests, can't rerun a single test by name, can't integrate with CI without parsing stdout, and duplicates setup boilerplate. The bloom filter tester (`tester_test_bloom_filter.py`) imports `pytest` anyway for `pytest.raises` (line 4) — so it's not actually framework-independent despite the convention's intent.

The **pytest** convention requires a dependency. It also hides diagnostic output behind flags — you won't see tree height or page counts unless you add `-s`. And it doesn't serve the "run this to see what the code does" educational goal at all.

## The Pattern Across the Codebase

The two conventions aren't always aligned. The bloom filter pair is instructive: `tester_test_bloom_filter.py` (125 lines) and `test_bloom_filter.py` (207 lines) cover largely the same test cases — no-false-negatives, FPR bounds, optimal parameters, counting filter, serialization, union, estimate count, edge cases, determinism. But `test_bloom_filter.py` adds boundary tests the tester skips: `test_explicit_parameters` (line 63), `test_counting_double_add_remove` (line 89), `test_single_item` (line 140), `test_duplicate_add` (line 146), `test_union_incompatible` (line 118), and `test_bit_array_density` (line 172).

The tester files are the "proof it works" layer. The test files are the "prove it's correct" layer.

## Topics to Explore

- [file] `change-data-capture/test_cdc.py` — Uses `@pytest.fixture` for both database and CDC pipeline setup — best example of fixture reuse that tester files can't replicate
- [file] `log-structured-hash-table/test_bitcask.py` — Another fixture-heavy test file; compare with its tester counterpart to see the setup duplication tradeoff
- [file] `write-skew-detection/tester_test_ssi.py` — Has a more sophisticated runner (lines 190–213) with pass/fail counting and exception capture — the tester convention pushed toward its limits
- [function] `b-tree-storage-engine/test_btree.py:test_wal_uncommitted_entries` — A white-box test that imports WAL internals and manually writes entries — the kind of test only possible in the pytest layer
- [general] `pytest-dependency-in-testers` — `tester_test_bloom_filter.py` imports pytest despite being a "framework-independent" tester — investigate which other tester files leak this dependency

## Beliefs

- `tester-files-are-self-contained-runners` — Every `tester_test_*.py` has an `if __name__ == '__main__':` block that calls all test functions sequentially and prints human-readable pass/fail output
- `test-files-cover-more-cases` — `test_*.py` files consistently have more test functions than their `tester_test_*.py` counterparts, covering internal/white-box scenarios the testers skip
- `tester-bloom-filter-leaks-pytest` — `tester_test_bloom_filter.py` imports `pytest` (line 4) for `pytest.raises`, breaking the framework-independence convention that `tester_test_*.py` files otherwise follow
- `fixture-usage-is-sparse` — Only 3 of the `test_*.py` files use `@pytest.fixture` (test_cdc, test_map_side_joins, test_bitcask); most pytest files still use inline setup like their tester counterparts
- `tester-print-includes-diagnostics` — Tester files embed runtime metrics in their output (tree height, page counts, pass/fail ratios) that pytest's structured reporting doesn't surface without `-s`

