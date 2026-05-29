# Topic: Why the repo maintains both `test_*.py` (pytest) and `tester_test_*.py` (standalone) files across all implementations

**Date:** 2026-05-29
**Time:** 06:25

## Why Both `test_*.py` and `tester_test_*.py` Exist

The repo maintains two parallel test suites per implementation, each serving a different execution model.

### `test_*.py` — Pytest-integrated test suites

These are conventional pytest files: they `import pytest` (24 matches across the repo), use bare `assert` statements, and rely on pytest's test runner for discovery, execution, and reporting. They contain richer coverage — `bloom-filter/test_bloom_filter.py` is 207 lines with additional edge cases like random-string testing, while the corresponding tester file is 125 lines. They have **no** `if __name__` blocks, meaning they're designed exclusively to run via `pytest`.

### `tester_test_*.py` — Standalone verification scripts

Every one of these files includes an `if __name__ == "__main__":` block (13 matches found). They're designed to run as plain Python scripts: `python tester_test_btree.py`. The critical marker is the `print("test_basic_put_get PASSED")` pattern visible at `b-tree-storage-engine/tester_test_btree.py:49` — this is **output-based validation**, where a runner checks stdout for "PASSED" strings rather than relying on pytest's exit code and structured reporting.

The naming is intentional: `tester_test_*.py` does **not** match pytest's default discovery pattern (`test_*.py` or `*_test.py`), so these files are invisible to `pytest` unless explicitly targeted. This prevents double-counting when running `pytest` across the repo.

### Why the duplication?

The two suites target different consumers:

| Dimension | `test_*.py` | `tester_test_*.py` |
|-----------|-------------|---------------------|
| Runner | `pytest` | `python` directly |
| Dependencies | Requires pytest | Some import pytest for fixtures, many don't |
| Output | Pytest report (exit code) | Stdout `PASSED`/`FAILED` lines |
| Discovery | Auto-discovered by pytest | Must be invoked explicitly |
| Coverage | Broader, more edge cases | Focused on core invariants |

The `tester_test_*.py` files act as a **zero-dependency validation harness** — they verify that each implementation meets its behavioral contract without requiring any test framework infrastructure. This is the kind of design you see in educational repos or automated grading systems, where the verifier must run in minimal environments. The pytest files then layer on top with more thorough coverage for development use.

Note that the boundary isn't perfectly clean: several tester files (`bloom-filter/tester_test_bloom_filter.py:4`, `fencing-tokens/tester_test_fencing_tokens.py:2`, `consistent-hashing/tester_test_consistent_hashing.py:3`) still `import pytest` for markers or fixtures, suggesting they evolved from the pytest files or were written pytest-compatible as a convenience even though their primary use case is standalone execution.

---

## Topics to Explore

- [general] `tester-runner-system` — What orchestration script or CI job invokes the `tester_test_*.py` files and parses their stdout PASSED/FAILED output? The runner is the missing piece that explains why the output-based format exists.
- [file] `b-tree-storage-engine/tester_test_btree.py` — Read the full `if __name__ == "__main__":` block (line 190) to see how tests are collected and run standalone, and whether failures are aggregated.
- [diff] `test_btree.py vs tester_test_btree.py` — Side-by-side comparison reveals which tests are unique to each file — the pytest file has 100 more lines, so understanding what's added (and what's deliberately excluded from the tester) clarifies the contract boundary.
- [file] `unbundled-database/tester_test_unbundled_database.py` — At 555 lines (largest tester file by far), this is the most complex standalone test and likely shows advanced patterns the simpler tester files don't need.
- [general] `pytest-configuration` — Check for `pyproject.toml`, `pytest.ini`, or `conftest.py` at the repo root to confirm whether `tester_test_*.py` files are explicitly excluded from pytest collection.

---

## Beliefs

- `tester-files-are-standalone-runnable` — Every `tester_test_*.py` file contains an `if __name__ == "__main__":` block, making it executable via `python` without pytest.
- `tester-naming-avoids-pytest-discovery` — The `tester_test_*.py` prefix does not match pytest's default collection patterns (`test_*.py` / `*_test.py`), so these files are excluded from `pytest` runs unless explicitly specified.
- `tester-files-use-stdout-validation` — Tester files print `"test_name PASSED"` to stdout, indicating an output-parsing runner rather than pytest's exit-code-based reporting.
- `pytest-files-have-broader-coverage` — The `test_*.py` files are consistently longer than their `tester_test_*.py` counterparts (e.g., 207 vs 125 lines for bloom-filter, 300 vs 200 for b-tree), containing additional edge case tests.
- `tester-pytest-boundary-is-leaky` — At least 6 `tester_test_*.py` files import pytest despite being designed for standalone execution, indicating the two suites share lineage rather than being fully independent.

