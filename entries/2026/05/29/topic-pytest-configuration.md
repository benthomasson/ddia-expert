# Topic: Check for `pyproject.toml`, `pytest.ini`, or `conftest.py` at the repo root to confirm whether `tester_test_*.py` files are explicitly excluded from pytest collection.

**Date:** 2026-05-29
**Time:** 08:08

## No Pytest Configuration Exists — `tester_test_*.py` Files Are Not Excluded

The repository has **no pytest configuration at all**. None of the standard configuration files exist:

| File | Status |
|------|--------|
| `pyproject.toml` | Not found |
| `pytest.ini` | Not found |
| `setup.cfg` | Not found |
| `conftest.py` (root) | Not found |

Without any of these files, pytest falls back entirely to its **default collection rules**: it recurses from the invocation directory, collects files matching `test_*.py` and `*_test.py`, and runs functions/classes matching `test_*`/`Test*`.

### What this means for `tester_test_*.py`

Files named `tester_test_*.py` **would be collected by pytest** under default rules, because they match the `*_test.py` glob pattern. There is no `collect_ignore`, `norecursedirs`, `testpaths`, or `python_files` setting anywhere in the repo to override this — the grep for these directives across all files returned zero matches.

The `.gitignore` (4 lines) only excludes `__pycache__/`, `*.pyc`, and `.pytest_cache/` — standard cache artifacts, nothing test-related.

### Bottom line

If `tester_test_*.py` files exist in this repo and someone runs `pytest` from the root, **pytest will discover and attempt to run them**. There is no exclusion mechanism in place. If these files are intended as test infrastructure (not test suites themselves), this is a latent problem — they'll be collected and likely fail or produce confusing results.

---

## Topics to Explore

- [general] `tester-test-file-locations` — Find where `tester_test_*.py` files actually live in the repo and whether they contain runnable test functions or are infrastructure/harness code
- [general] `pytest-default-collection` — Understand pytest's default discovery rules (`test_*.py`, `*_test.py`) to know exactly which files get swept up without configuration
- [repo] `/Users/ben/git/ddia-implementations` — Survey the top-level directory structure to understand why no `pyproject.toml` or test configuration exists (is this intentional, or was setup never completed?)
- [general] `conftest-hierarchy` — Check whether any `conftest.py` files exist in subdirectories that might locally configure collection, even though the root has none
- [file] `.gitignore` — The minimal 4-line gitignore suggests this repo may be early-stage; check if other standard project scaffolding is also missing

## Beliefs

- `no-pytest-config-exists` — The repository contains no `pyproject.toml`, `pytest.ini`, `setup.cfg`, or root `conftest.py`; pytest runs with pure default settings
- `tester-test-files-not-excluded` — No `collect_ignore`, `norecursedirs`, `testpaths`, or `python_files` directive exists anywhere in the repo, so `tester_test_*.py` files will be collected by pytest's default `*_test.py` glob
- `gitignore-is-minimal` — `.gitignore` contains only three entries (`__pycache__/`, `*.pyc`, `.pytest_cache/`) with no test-related exclusions
- `default-pytest-collection-applies` — Without configuration overrides, pytest will recurse all directories (except hidden and common virtualenv dirs) and collect any file matching `test_*.py` or `*_test.py`

