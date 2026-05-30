# Topic: Understand pytest's default discovery rules (`test_*.py`, `*_test.py`) to know exactly which files get swept up without configuration

**Date:** 2026-05-29
**Time:** 11:36

# Pytest Discovery Rules in `ddia-implementations`

## How pytest finds tests (without configuration)

This repo has **zero pytest configuration** — no `pytest.ini`, no `pyproject.toml`, no `setup.cfg`, no `conftest.py` at any level. That means pytest runs entirely on its default discovery rules:

1. **File collection**: Starting from the invocation directory, pytest recursively finds files matching `test_*.py` or `*_test.py`.
2. **Inside those files**: it collects functions named `test_*` and classes named `Test*` (with methods named `test_*`).
3. **Directories**: it recurses into all subdirectories unless they match `norecursedirs` defaults (`.git`, `__pycache__`, `.tox`, etc.).

## What actually gets swept up

This is where it gets interesting. The repo has **two parallel sets of test files** per module:

| Pattern | Example | Collected? |
|---------|---------|------------|
| `test_*.py` | `bloom-filter/test_bloom_filter.py` | **Yes** — matches `test_*.py` |
| `tester_test_*.py` | `bloom-filter/tester_test_bloom_filter.py` | **No** — `tester_test_` does not match `test_*` or `*_test` |

The `tester_test_*.py` files are invisible to default pytest discovery. The prefix `tester_` breaks the `test_*.py` glob — pytest requires the filename to *start with* `test_` or *end with* `_test` (before `.py`). `tester_test_bloom_filter.py` starts with `tester_`, not `test_`.

### Concrete inventory

**Collected by default** (the `test_*.py` files):
- `fencing-tokens/test_fencing_tokens.py` — 5 test classes (`TestLockAcquisition`, `TestLockRenewal`, `TestFencedServer`, `TestScenarios`, `TestClientEdgeCases`)
- `gossip-protocol/test_gossip_protocol.py` — 10 standalone test functions
- `merkle-tree/test_merkle_tree.py` — 10 standalone test functions
- `bloom-filter/test_bloom_filter.py` — 5 standalone test functions

**Not collected by default** (the `tester_test_*.py` files):
- `fencing-tokens/tester_test_fencing_tokens.py` — 2 test classes
- `gossip-protocol/tester_test_gossip_protocol.py` — 12 test functions (includes extras like `test_join_does_not_use_stale_timestamp` at line 180 and `test_new_node_via_gossip_gets_current_timestamp` at line 201 that don't exist in the `test_` version)
- `bloom-filter/tester_test_bloom_filter.py` — 11 test functions (significantly more coverage than the `test_` version's 5)
- `write-skew-detection/tester_test_ssi.py` — 5 test functions (this module has **no** `test_*.py` counterpart, so its tests are entirely invisible to default discovery)

### The practical consequence

Running bare `pytest` from the repo root collects ~30 test items from the `test_*.py` files. The `tester_test_*.py` files contain ~40+ additional test items — including the *only* tests for `write-skew-detection` — that silently don't run. The `tester_test_` files also tend to have broader coverage (e.g., bloom-filter's tester version has 11 tests vs. 5 in the default-discovered version, covering serialization, union, scalable filters, and counting filters).

To run the tester files, you'd need either explicit paths (`pytest bloom-filter/tester_test_bloom_filter.py`) or a `pytest.ini` with a custom `python_files` glob like `python_files = test_*.py tester_test_*.py`.

## Topics to Explore

- [file] `write-skew-detection/tester_test_ssi.py` — The only test file for write-skew-detection, completely invisible to default pytest discovery
- [file] `bloom-filter/tester_test_bloom_filter.py` — Compare against `test_bloom_filter.py` to understand why two parallel test suites exist and what the tester version covers that the default-discovered one doesn't
- [general] `tester-vs-test-naming-convention` — Investigate whether the `tester_test_` prefix is intentional (e.g., generated tests vs. hand-written) or accidental, and whether a `conftest.py` or `pytest.ini` was intended but never created
- [general] `pytest-collection-customization` — Explore `python_files`, `python_classes`, and `python_functions` ini-options that would unify both test sets under a single `pytest` invocation
- [diff] `gossip-protocol/test_gossip_protocol.py` — Diff against `tester_test_gossip_protocol.py` to see which tests are unique to each version (the tester version has 2 extra tests covering stale timestamp and gossip-discovered nodes)

## Beliefs

- `no-pytest-config-exists` — The ddia-implementations repo has no pytest.ini, pyproject.toml, setup.cfg, or conftest.py, so pytest runs entirely on default discovery rules
- `tester-test-files-not-discovered` — Files named `tester_test_*.py` are not collected by pytest's default `test_*.py` / `*_test.py` globs, silently excluding ~40% of the test suite
- `write-skew-has-no-default-tests` — The `write-skew-detection` module's tests exist only in `tester_test_ssi.py`, making it entirely untested under default pytest invocation
- `tester-versions-have-broader-coverage` — The `tester_test_*.py` files consistently contain more test functions than their `test_*.py` counterparts (e.g., bloom-filter: 11 vs. 5, gossip-protocol: 12 vs. 10)

