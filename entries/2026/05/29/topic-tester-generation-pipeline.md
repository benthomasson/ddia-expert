# Topic: How the code-expert workflow generates `tester_test_*.py` files from specs, and whether they're regenerated when specs change

**Date:** 2026-05-29
**Time:** 10:20

## How `tester_test_*.py` Files Relate to the Code-Expert Workflow

### Short answer: the observations show no generation mechanism

The search for any code that generates `tester_test_*.py` files came up empty across every angle:

| What was searched | Result |
|---|---|
| Grep for `tester_test` in generation code | **0 matches** |
| Grep for `generated\|auto-generated\|DO NOT EDIT` in tester files | **0 matches** |
| `.code-expert/` directory contents | Only `topics.json` — no templates, specs, or generator config |
| CLAUDE.md workflow steps | Scan → explore → explain → propose beliefs → review → derive. **No test generation step.** |

### What the files actually look like

The `tester_test_*.py` files are **parallel, simplified versions** of the hand-written `test_*.py` files. Comparing the bloom filter pair:

- `bloom-filter/test_bloom_filter.py` — 207 lines, 16+ tests, includes edge cases like `test_duplicate_add`, `test_union_incompatible`, `test_counting_double_add_remove`
- `bloom-filter/tester_test_bloom_filter.py` — 125 lines, 11 tests, covers the same core scenarios but omits edge cases and uses more compact style

The tester version reads like a **distilled spec**: numbered comments (`# 1. No false negatives`, `# 2. FPR within 2x of target`) map to the key behavioral properties, and the tests are stripped to minimum assertions. But there are no markers, no generation comments, and no tooling to produce or regenerate them.

### What's missing from the observations

To definitively answer whether these files are generated and whether they track spec changes, you'd need to find:

1. **The generator itself** — search the code-expert skill's source code (likely in a plugin or CLI tool outside this repo) for `tester_test` or test-generation logic
2. **Spec files** — the "specs" these are supposedly generated from; none were found in `ddia-implementations` or `.code-expert/`
3. **Git history** — `git log --diff-filter=A -- '*/tester_test_*.py'` would reveal whether these files were added by a human commit or a tool-automated one
4. **The code-expert skill implementation** — the CLAUDE.md references commands like `code-expert explain file` and `code-expert explore`, but the skill's source code wasn't searched

### Most likely explanation

The `tester_test_*.py` files were **generated once by an LLM-driven process** (likely during a `code-expert explain` or exploration session) as a way to codify understood behavior, then committed as static files. They are **not automatically regenerated** when the implementation changes — there is no watcher, no spec-to-test pipeline, and no regeneration command in the documented workflow.

---

## Topics to Explore

- [general] `code-expert-skill-source` — Find the code-expert skill's implementation to see if it has hidden test-generation logic not documented in CLAUDE.md
- [diff] `git log -- '*/tester_test_*.py'` — Check commit history to see how and when tester_test files were first introduced
- [file] `bloom-filter/test_bloom_filter.py` — Compare the hand-written tests against the tester version to understand what the distillation drops and keeps
- [file] `.code-expert/topics.json` — The only config file in the tool state directory; may reveal how the exploration queue relates to test file creation
- [general] `tester-test-staleness` — Diff each `tester_test_*.py` against its `test_*.py` sibling to find behavioral divergences that indicate the tester version has gone stale

---

## Beliefs

- `tester-tests-not-auto-regenerated` — `tester_test_*.py` files contain no auto-generation markers and no regeneration tooling exists in the documented code-expert workflow
- `tester-tests-are-distilled-specs` — Each `tester_test_*.py` is a simplified subset of its corresponding `test_*.py`, organized by numbered behavioral properties rather than exhaustive edge cases
- `code-expert-has-no-test-generation-step` — The code-expert workflow (scan → explore → explain → propose-beliefs → review → derive) does not include a test file generation or update step
- `tester-and-original-tests-coexist` — Both `tester_test_*.py` and `test_*.py` exist side-by-side in each module directory, testing the same implementation with different coverage depth

