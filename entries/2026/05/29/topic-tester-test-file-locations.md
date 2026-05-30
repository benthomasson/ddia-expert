# Topic: Find where `tester_test_*.py` files actually live in the repo and whether they contain runnable test functions or are infrastructure/harness code

**Date:** 2026-05-29
**Time:** 11:36

# Where `tester_test_*.py` Files Live and What They Are

## Location Pattern

Every `tester_test_*.py` file lives **one level deep** inside a module-specific directory in the target repo (`/Users/ben/git/ddia-implementations`). The naming convention is `<module-dir>/tester_test_<module>.py`:

| Directory | Test File |
|-----------|-----------|
| `b-tree-storage-engine/` | `tester_test_btree.py` |
| `bloom-filter/` | `tester_test_bloom_filter.py` |
| `change-data-capture/` | `tester_test_cdc.py` |
| `write-ahead-log/` | `tester_test_wal.py` |
| `gossip-protocol/` | `tester_test_gossip_protocol.py` |
| `write-skew-detection/` | `tester_test_ssi.py` |
| `fencing-tokens/` | `tester_test_fencing_tokens.py` |
| `leader-follower-replication/` | `tester_test_replication.py` |
| `secondary-index-partitioning/` | `tester_test_secondary_index_partitioning.py` |
| `sstable-and-compaction/` | `tester_test_sstable.py` |
| `unbundled-database/` | `tester_test_unbundled_database.py` |
| ... | (and more — 195 `def test_` functions total across all files) |

There is no centralized `tests/` directory. Each module is self-contained: implementation code and its test file sit side by side.

## They Are Real, Runnable Tests — Not Harness Code

These files are **genuine test suites**, not infrastructure. Three pieces of evidence:

### 1. They contain `def test_*` functions that assert behavior

Every file is packed with standard test functions that import the module under test, exercise its API, and assert results. For example, `b-tree-storage-engine/tester_test_btree.py:8` defines `test_basic_put_get()` which creates a `BTree`, inserts data, and asserts lookups return the right values. The grep found **195 such functions** across all `tester_test_*.py` files.

### 2. They use two execution modes

**pytest-compatible**: Some files use `pytest` fixtures and `pytest.raises` (e.g., `change-data-capture/tester_test_cdc.py:12` defines a `@pytest.fixture`, and `bloom-filter/tester_test_bloom_filter.py:4` imports `pytest`). These run via `pytest`.

**Standalone `__main__` runner**: Others skip pytest entirely and include a `if __name__ == '__main__':` block that calls each test function in sequence, printing "PASSED" per test. See `b-tree-storage-engine/tester_test_btree.py:192` and `write-ahead-log/tester_test_wal.py:172`. These run with plain `python tester_test_*.py`.

### 3. They import only the module under test — no shared harness

The import grep (`grep_tester_imports`) shows each file imports from its own module (`from btree import BTree`, `from wal import WriteAheadLog`, etc.) plus standard library utilities (`tempfile`, `os`). The `conftest` grep returned **zero matches** — there is no shared `conftest.py` or test harness that references these files. Each test file is fully self-contained.

### 4. Some use class-based grouping

Files like `fencing-tokens/tester_test_fencing_tokens.py` and `leader-follower-replication/tester_test_replication.py` organize tests into classes (`TestLockBasics`, `TestFailover`, etc.) — 63 class definitions total. These are standard pytest test classes, not base classes or mixins.

## Why the `tester_` prefix?

The prefix is unusual — standard pytest discovery expects `test_*.py`. This suggests the files were generated or templated by an external tool (likely the code-expert pipeline referenced in `CLAUDE.md`), and the `tester_` prefix distinguishes them from any hand-written `test_*.py` files. To run them with pytest you'd need either `pytest tester_test_btree.py` (explicit path) or a `pyproject.toml`/`conftest.py` that adjusts `python_files` discovery.

## Topics to Explore

- [general] `tester-pytest-discovery` — Check whether a `pyproject.toml` or `pytest.ini` configures `python_files = tester_test_*.py` to allow standard `pytest` discovery of these files
- [file] `b-tree-storage-engine/tester_test_btree.py` — A clean example of the standalone runner pattern — no pytest dependency, pure `assert` + `__main__` block
- [file] `change-data-capture/tester_test_cdc.py` — Demonstrates the pytest-fixture pattern with `@pytest.fixture` and `pytest.raises`, contrast with the standalone runner files
- [function] `leader-follower-replication/tester_test_replication.py:TestFailover` — Shows how class-based test grouping models a complex distributed-systems scenario
- [general] `test-generation-origin` — Investigate how these `tester_test_*.py` files were originally generated (the code-expert pipeline?) and whether they're meant to be edited by hand

## Beliefs

- `tester-test-files-colocated` — Every `tester_test_*.py` file lives in the same directory as the module it tests, not in a centralized `tests/` folder
- `tester-test-no-shared-harness` — There is no shared conftest, base class, or test harness across `tester_test_*.py` files; each is fully self-contained
- `tester-test-dual-runner` — Some `tester_test_*.py` files are pytest-only (using fixtures/raises), while others include a `__main__` block for standalone execution — the two patterns coexist across modules
- `tester-test-195-functions` — The repo contains 195 `def test_*` functions and 63 test classes spread across all `tester_test_*.py` files
- `tester-prefix-non-standard` — The `tester_` prefix means these files won't be discovered by pytest's default `test_*.py` glob without explicit configuration

