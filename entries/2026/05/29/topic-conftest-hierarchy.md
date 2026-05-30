# Topic: Check whether any `conftest.py` files exist in subdirectories that might locally configure collection, even though the root has none

**Date:** 2026-05-29
**Time:** 11:39

## No `conftest.py` Files Exist Anywhere in the Repository

The observations conclusively show that **zero `conftest.py` files exist** — not at the repo root, and not in any subdirectory.

### Evidence

1. **Root check** (`check_root_conftest`): Returned `"File not found: conftest.py"` — no root-level conftest.

2. **Full grep** (`find_conftest_files`): Searched the entire codebase for the string `conftest`. All 11 matches are in `.code-expert/topics.json` — the topic queue that *tracks this very question*. No actual `conftest.py` files surfaced.

3. **Directory listing** (`list_all_conftest`): The 147 entries across all module directories show a consistent per-module structure: an implementation file, a `test_*.py` file, and sometimes a `tester_test_*.py` file. No `conftest.py` appears in any subdirectory.

4. **No pytest configuration at all** (`find_pytest_ini_or_config`, `check_test_collection_config`): No `pyproject.toml`, `pytest.ini`, `setup.cfg`, or `tox.ini` exists. No `collect_ignore`, `norecursedirs`, or `testpaths` directives are configured anywhere.

### What This Means

Each module directory (e.g., `bloom-filter/`, `raft-consensus/`) operates as a **fully self-contained unit** with no shared test fixtures, no inherited configuration, and no pytest collection customization. When you run `pytest` from any module directory, it uses pure default discovery rules (`test_*.py` files, `Test*` classes, `test_*` functions) with no local overrides.

This also means there is no mechanism to:
- Share fixtures across modules (e.g., a common temp-directory fixture)
- Exclude the `tester_test_*.py` files from pytest collection via conftest
- Register custom markers or plugins at any level

The `tester_test_*.py` files **will** be collected by pytest's default rules (they match `test_*.py` glob after the `tester_` prefix — actually, they match `*_test_*.py` which is not a default pattern, but the files contain `test_` prefixed functions that would be collected if pytest enters the file). The lack of any conftest or config to exclude them is architecturally significant.

---

## Topics to Explore

- [general] `pytest-default-collection` — Understand exactly which of the `tester_test_*.py` files pytest would collect by default, given there's no config to include or exclude them
- [general] `tester-vs-pytest-test-duality` — Why the repo maintains both `test_*.py` (pytest) and `tester_test_*.py` (standalone) files across modules, and whether the duality is intentional
- [general] `pytest-dependency-in-testers` — Some `tester_test_*.py` files (e.g., `tester_test_bloom_filter.py`) import pytest despite being "framework-independent" — investigate the boundary blur (topics.json:5275)
- [file] `fencing-tokens/test_fencing_tokens.py` — One of the few files that explicitly `import pytest` (line 2) — check whether it uses fixtures or markers that would normally live in a conftest
- [general] `module-isolation-architecture` — Survey whether any modules have their own `requirements.txt` or `pyproject.toml`, confirming the fully-decoupled design pattern

## Beliefs

- `no-conftest-anywhere` — The ddia-implementations repo contains zero `conftest.py` files at any directory level — root or subdirectory
- `no-pytest-config-files` — No `pyproject.toml`, `pytest.ini`, `setup.cfg`, or `tox.ini` exists in the repo, so pytest runs with pure default settings everywhere
- `no-collection-customization` — No `collect_ignore`, `collect_ignore_glob`, `norecursedirs`, or `testpaths` directives exist anywhere in the repo
- `modules-share-no-fixtures` — Without any conftest hierarchy, there is no mechanism for modules to share test fixtures, markers, or plugins across directory boundaries
- `tester-files-not-excluded` — The `tester_test_*.py` files have no conftest or configuration-based exclusion from pytest collection

