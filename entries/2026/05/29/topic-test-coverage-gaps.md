# Topic: Whether tester files ever test things the developer suite doesn't, or if the developer suite is always a strict superset

**Date:** 2026-05-29
**Time:** 10:24

# Whether Tester Files Ever Test Things the Developer Suite Doesn't

## The Observations Were Insufficient

The provided observations show 0 matches for `def test_` across all files, which contradicts the file content also shown (e.g., `tester_test_btree.py` clearly contains `def test_basic_put_get()` at line 8). The grep results appear to have failed silently — possibly a path or permission issue. To answer this question properly, I pulled from the project's existing code exploration entries.

## Answer: The Developer Suite Is NOT a Strict Superset

The two suites test **different dimensions**, and there are concrete cases where tester files cover ground the developer suite does not.

### What the developer suite adds (the expected direction)

The dev `test_*.py` files are consistently longer — 207 vs 125 lines for bloom-filter, ~300 vs ~200 for b-tree — and go deeper into implementation internals:

- `test_btree.py` tests WAL crash recovery (line 119), uncommitted WAL entries (line 144), and delete freeing empty leaves (line 171). The tester file never touches any of these.
- `test_consistent_hashing.py` includes `test_duplicate_add_idempotent` (line 108), `test_ring_position_valid_range` (line 114), `test_scalability` (line 120), and `test_vnode_count_affects_balance` (line 153) — none appear in the tester.
- Dev tests import internal types (`WAL`, `_serialize_leaf`, `HEADER_FMT`) for state injection; tester files never import internals.

### What only the tester files test (the surprising direction)

**1. Spec-example compliance tests.** `unbundled-database/test_tester_validation.py` contains `TestSpecExample.test_full_spec_example` — a single test that runs the exact "Example Usage" from the module's design spec end-to-end. This is a **living executable spec**: if the spec example ever diverges from the implementation, this test breaks first. The developer suite doesn't replicate this spec-verbatim walkthrough.

**2. Catch-up/rebuild equivalence checks.** `test_tester_validation.py:test_rebuild_after_catch_up_matches_live` compares two independent derived systems via `get_state()`, asserting that snapshot-based catch-up and full event replay produce identical output. This is a commutativity invariant that the dev suite may exercise implicitly but doesn't assert explicitly as a cross-path equivalence.

**3. Full lifecycle integration scenarios.** `leaderless-replication/test_dynamo_tester.py:test_spec_example_full` walks through an entire Dynamo lifecycle — write → read → node failure → degraded quorum write → recovery → read repair → quorum failure → anti-entropy — in a single test. The dev suite typically tests these behaviors in isolation.

**4. Stdout-based pass reporting as a different failure mode.** Every tester file ends each test with `print("test_xxx PASSED")` (e.g., `tester_test_btree.py:46, 65, 83`). This isn't just reporting — it's a secondary assertion mechanism. If an assert fails, the corresponding PASSED line never prints, making silence a failure signal. The developer suite relies entirely on pytest's exit code.

### The Relationship Is Overlapping, Not Hierarchical

The suites test the same module but from different vantage points:

| Dimension | Tester files | Developer files |
|-----------|-------------|-----------------|
| **Scope** | Public API contract | Public API + internals |
| **Source of truth** | Spec requirements | Implementation knowledge |
| **Unique coverage** | Spec-example compliance, cross-path equivalence | Crash recovery, internal state, edge cases |
| **Imports** | Only public symbols | Public + internal types |

The tester files are a **correctness gate** ("does the implementation match the spec?"). The dev files are a **robustness layer** ("does the implementation handle crashes and edge cases?"). Neither is a strict superset of the other.

## Topics to Explore

- [file] `unbundled-database/test_tester_validation.py` — The clearest example of tester-exclusive coverage: `TestSpecExample` and `TestEdgeCases` test invariants the dev suite doesn't replicate
- [file] `leaderless-replication/test_dynamo_tester.py` — Full lifecycle integration test (`test_spec_example_full`) that exercises the entire Dynamo protocol in one scenario
- [file] `b-tree-storage-engine/test_btree.py` — Lines 119-183 show the opposite direction: WAL recovery and internal state tests that only the dev suite covers
- [general] `tester-staleness-risk` — Since tester files are generated once and not auto-regenerated, they can drift from both the spec and the dev tests over time
- [function] `unbundled-database/test_tester_validation.py:TestEdgeCases.test_rebuild_after_catch_up_matches_live` — Cross-path equivalence assertion that only exists in the tester layer

## Beliefs

- `tester-dev-suites-overlap-not-hierarchical` — The tester and developer test suites have overlapping but non-hierarchical coverage: tester files uniquely test spec-example compliance and cross-path equivalence, while dev files uniquely test crash recovery and internal state
- `tester-spec-compliance-is-unique` — `TestSpecExample` tests in tester files replay the exact spec usage example verbatim, a form of drift detection that developer test suites do not replicate
- `tester-files-never-import-internals` — Tester test files import only the public API (e.g., `from btree import BTree`), while developer test files import internal types like `WAL`, `_serialize_leaf`, and `HEADER_FMT`
- `dev-suite-longer-but-not-superset` — Developer `test_*.py` files are consistently longer (207 vs 125 lines for bloom-filter) with more edge cases, but they do not subsume the tester files' spec-conformance and lifecycle integration tests

