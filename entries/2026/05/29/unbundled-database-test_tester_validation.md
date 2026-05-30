# File: unbundled-database/test_tester_validation.py

**Date:** 2026-05-29
**Time:** 10:21

I'll work from what's available in the file content provided.

# `unbundled-database/test_tester_validation.py`

## Purpose

This is the **tester-stage validation suite** — a second layer of tests that sits above the primary test file (`test_unbundled_database.py`). Its job is twofold:

1. **Spec compliance**: `TestSpecExample` runs the exact "Example Usage" scenario from the module's design spec end-to-end, serving as a living executable specification. If the spec example ever diverges from the implementation, this test breaks.
2. **Edge-case coverage**: `TestEdgeCases` captures behaviors from reviewer notes and spec constraints that the primary tests might not cover — persistence roundtrips, empty inputs, internal state leaks on rebuild, catch-up vs. rebuild equivalence.

This naming convention (`test_tester_validation.py`) appears throughout the repo — it's the pattern for a "tester" layer that validates the test suite's own contracts against the spec, distinct from the `tester_test_*.py` files that are generated test harnesses.

## Key Components

### `TestSpecExample`

A single test method, `test_full_spec_example`, that exercises the entire `UnbundledDatabase` API in one linear flow:

- **Write path**: `put`, `delete`, `flush`
- **Read path**: `get`, `query_index`, `mv.query`, `fts.search`, `fts.search_all`
- **Derived systems**: `SecondaryIndex`, `MaterializedView`, `FullTextSearch` — added via `add_derived_system`
- **Lifecycle**: catch-up (`catch_up=True`), rebuild (`rebuild_system`), lag monitoring (`get_lag`), pipeline introspection (`get_pipeline_state`), storage rebuild from WAL (`storage.rebuild`)

This test is a **contract test** — it asserts exact counts, exact field values, and exact operation types (`event.operation == "update"`). Changing any behavior in the implementation will fail here before it fails anywhere else.

### `TestEdgeCases`

Seven focused tests, each targeting a specific invariant:

| Test | What it guards |
|------|---------------|
| `test_rebuild_after_catch_up_matches_live` | Catch-up via snapshot and rebuild via event replay produce identical derived state |
| `test_lsn_sequential_no_gaps` | LSNs are 1-indexed and strictly sequential |
| `test_nested_dict_values` | Nested dicts in records survive write/read without corruption |
| `test_empty_search_all` | `search_all([])` returns `[]`, not an error |
| `test_storage_engine_rebuild_clears_old_state` | `rebuild` wipes phantom/stale state before replaying WAL |
| `test_add_derived_system_no_catch_up` | `catch_up=False` means the system starts empty, only seeing future writes |
| `test_wal_persistence_roundtrip` | WAL written to disk can fully reconstruct storage in a new `UnbundledDatabase` instance |

## Patterns

**Spec-as-test**: `TestSpecExample` mirrors the spec's example usage verbatim. This is a deliberate choice — it prevents spec/implementation drift and serves as onboarding documentation for anyone reading the spec first.

**State-equivalence assertion**: `test_rebuild_after_catch_up_matches_live` compares two independent derived systems via `get_state()`, asserting that two different code paths (snapshot-based catch-up vs. full event replay) produce identical output. This is a classic **commutativity check** for eventually-consistent systems.

**Injection-then-rebuild**: `test_storage_engine_rebuild_clears_old_state` manually injects `db.storage._data["phantom"]` to simulate corrupted or stale state, then asserts `rebuild` eliminates it. This pattern of polluting internal state to verify cleanup is common in storage-engine tests.

**Filesystem isolation**: `test_wal_persistence_roundtrip` uses `tempfile.TemporaryDirectory` for disk-backed WAL tests, ensuring no cross-test filesystem contamination.

## Dependencies

**Imports from `unbundled_database`:**
- `WALEntry`, `CDCEvent` — data classes for the write-ahead log and change-data-capture stream
- `WriteAheadLog`, `StorageEngine`, `CDCStream` — core infrastructure
- `SecondaryIndex`, `MaterializedView`, `FullTextSearch` — derived system implementations
- `UnbundledDatabase` — the orchestrator that wires everything together

**External:** `pytest`, `os`, `tempfile`

**Imported by:** Nothing — this is a leaf test file. It's consumed by the test runner.

Note: `os` is imported but unused (likely a leftover from an earlier version of the persistence test).

## Flow

The test suite runs linearly within each test, but the underlying data flow in the system under test is:

```
put/delete → WAL append → CDCStream event → flush → derived systems consume events
```

`TestSpecExample.test_full_spec_example` walks through the full lifecycle:
1. Create database, register three derived systems
2. Insert 3 records, flush to propagate
3. Assert query/index/search results
4. Update a record, assert CDC event fields (`operation`, `old_value`, `new_value`)
5. Flush, verify derived systems reflect the update
6. Delete a record, verify cascading effects
7. Add a new derived system with `catch_up=True`, verify it sees existing data
8. Rebuild an existing system, verify event count
9. Insert with unflushed lag, assert lag > 0, flush, assert lag == 0
10. Inspect pipeline state for record count and derived system count
11. Rebuild storage from WAL, verify final state

## Invariants

- **LSN monotonicity**: LSNs are 1-indexed, sequential, gap-free (enforced by `test_lsn_sequential_no_gaps`)
- **Catch-up/rebuild equivalence**: Snapshot-based catch-up and full event replay must produce identical derived state
- **Rebuild atomicity**: `rebuild` must clear all prior state — no phantom records survive
- **Lag correctness**: After `flush`, all derived systems must have lag == 0
- **CDC event completeness**: Update events must carry both `old_value` and `new_value`
- **Pipeline state accuracy**: `storage_records` reflects live record count (3 after inserting 4 and deleting 1)

## Error Handling

This file contains no explicit error handling — it's a test suite that asserts happy-path invariants and edge-case behaviors. Failures surface as `AssertionError` via `pytest`. There are no `pytest.raises` blocks, meaning none of these tests verify error conditions (e.g., querying a nonexistent index, or double-deleting a key). That's a gap this file intentionally leaves to the primary test suite or the tester harness.

## Topics to Explore

- [file] `unbundled-database/unbundled_database.py` — The implementation under test; critical for understanding `flush`, `add_derived_system(catch_up=True)`, and `rebuild_system` internals
- [function] `unbundled-database/unbundled_database.py:UnbundledDatabase.flush` — The flush mechanism that propagates CDC events to derived systems; central to lag semantics
- [file] `unbundled-database/tester_test_unbundled_database.py` — The generated tester harness; compare its coverage with this validation suite to understand the two-layer testing strategy
- [general] `catch-up-vs-rebuild-equivalence` — The architectural guarantee that snapshot-based catch-up and full event replay produce identical state — a foundational property of log-based derived data systems (DDIA Chapter 11)
- [general] `tester-generation-pipeline` — How the `tester_test_*.py` files are generated across the repo and how they relate to validation files like this one

## Beliefs

- `unbundled-catchup-rebuild-equivalence` — Catch-up via `snapshot_and_stream` and rebuild via full CDC event replay must produce identical derived-system state
- `unbundled-lsn-sequential` — LSNs returned by `put`/`delete` are 1-indexed and strictly sequential with no gaps
- `unbundled-rebuild-clears-state` — `StorageEngine.rebuild(wal)` clears all existing data (including manually injected entries) before replaying WAL entries
- `unbundled-flush-zeroes-lag` — After `db.flush()`, `get_lag()` returns 0 for all derived systems
- `unbundled-cdc-update-carries-old-and-new` — CDC events for update operations include both `old_value` and `new_value` fields reflecting the prior and new record state

