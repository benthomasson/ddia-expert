# Topic: Whether `pyproject.toml`, `pytest.ini`, or `conftest.py` exists at the repo root (topics.json:1283 flags this as unresolved) — their absence would confirm pytest is per-directory, reinforcing why standalone testers exist

**Date:** 2026-05-29
**Time:** 11:32

# No Root-Level Pytest Configuration Exists

The repo root at `/Users/ben/git/ddia-implementations/` contains **zero configuration files** — no `pyproject.toml`, no `pytest.ini`, no `conftest.py`, no `setup.cfg`. The root is exclusively 37 directories, one per DDIA concept, plus a `.gitignore`. A grep for `[pytest]` or `[tool.pytest` across the entire repo returned zero matches, confirming there's no pytest configuration hiding in any file at any level.

This is architecturally significant. It means:

1. **Each module is a standalone island.** There is no unified test runner, no shared fixtures, no common conftest. You `cd bloom-filter && pytest` — you don't run `pytest` from the root and expect it to discover everything.

2. **The `tester_*.py` naming convention makes sense.** Look at `bloom-filter/tester_test_bloom_filter.py` — it imports directly from `bloom_filter` (line 6) with no package prefix, no `src.` path, no installed package. This only works when pytest's working directory is the module directory itself.

3. **The `.gitignore` confirms this pattern.** It ignores `__pycache__/`, `*.pyc`, and `.pytest_cache/` — artifacts that appear wherever pytest runs. Each module directory will grow its own `.pytest_cache/` independently.

4. **No shared test infrastructure exists.** There are no common fixtures, no shared test utilities, no base test classes. Each module defines its own test setup inline. The bloom filter tester, for example, constructs its own `BloomFilter` instances directly in each test function (lines 11-16, 22-27, etc.).

This design is intentional: each directory is a self-contained reference implementation that a reader can clone or copy in isolation. A root-level `pyproject.toml` would create coupling where the pedagogical goal is independence.

## Topics to Explore

- [general] `per-module-dependencies` — Whether any modules have their own `requirements.txt` or `pyproject.toml`, which would confirm the fully-decoupled pattern extends to dependency management
- [general] `tester-vs-test-naming` — Why some test files use `tester_test_*.py` naming instead of the standard `test_*.py` pattern, and whether pytest discovers them by default
- [file] `leaderless-replication/test_dynamo_tester.py` — A test file in a distributed-systems module; explore whether it requires runtime coordination (multiple processes, sockets) that would break under a unified test runner
- [general] `conftest-in-subdirectories` — Whether any individual module directories contain their own `conftest.py` with local fixtures
- [file] `bloom-filter/tester_test_bloom_filter.py` — Representative example of the standalone tester pattern; demonstrates how tests are structured without shared infrastructure

## Beliefs

- `no-root-pytest-config` — The ddia-implementations repo root contains no `pyproject.toml`, `pytest.ini`, `conftest.py`, or `setup.cfg`; there is zero pytest configuration at any level of the repo
- `modules-are-standalone-islands` — Each of the 37 module directories is a self-contained implementation with no cross-module imports or shared test infrastructure
- `pytest-must-run-per-directory` — Tests must be executed from within each module's directory (e.g., `cd bloom-filter && pytest`) because imports use bare module names with no package prefix
- `no-shared-fixtures-exist` — No `conftest.py` exists at the repo root or (based on available evidence) at any level, meaning each test file constructs its own setup inline

