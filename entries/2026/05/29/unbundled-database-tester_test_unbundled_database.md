# File: unbundled-database/tester_test_unbundled_database.py

**Date:** 2026-05-29
**Time:** 08:07

I'll explain the test file based on the content provided plus what's inferable from the API surface it exercises.

---

## Purpose

This is the **specification-as-tests** file for the unbundled database implementation — a reference implementation of the "unbundling the database" concept from DDIA Chapter 12. The idea: decompose a traditional monolithic database into composable subsystems (WAL, storage engine, CDC stream, secondary indexes, materialized views, full-text search), each independently maintainable and rebuildable from the shared change log.

The `tester_test_` prefix follows the repo's convention for the "golden" test suite — the one that defines the behavioral contract the implementation must satisfy (as opposed to `test_unbundled_database.py` which may contain implementation-level or developer-authored tests).

## Key Components

The file is organized as **16 numbered test classes**, each covering a distinct subsystem or integration concern:

| Class | What it specifies |
|-------|-------------------|
| `TestBasicPutGetDelete` | Core KV store semantics — put, get, update, delete, missing-key behavior |
| `TestWAL` | Write-ahead log: append, read_from offset, LSN tracking, persistence to JSONL |
| `TestCDCEvents` | Change data capture: insert/update/delete events with old_value/new_value |
| `TestSecondaryIndex` | Index maintenance through CDC — insert, update (re-index), delete (removal) |
| `TestMaterializedView` | Aggregate views (count, list) updated reactively via CDC |
| `TestFullTextSearch` | Inverted index with case-insensitive term/multi-term search |
| `TestCatchUp` | Late-joining derived systems replay the log to reach current state |
| `TestRebuild` | Rebuild a derived system from scratch and assert it matches live state |
| `TestStorageRebuild` | Storage engine can be reconstructed from the WAL alone |
| `TestLag` | Consumer lag: >0 after writes, 0 after flush |
| `TestPipelineIntrospection` | `get_pipeline_state()` returns WAL size, record count, CDC events, per-system lag |
| `TestCDCOldValue` | Dedicated coverage for old_value semantics across operation types |
| `TestDeleteCascade` | A single delete propagates correctly through index + view + FTS simultaneously |
| `TestMultipleConsumers` | Independent CDC subscribers; unsubscribing one doesn't affect others |
| `TestLogTruncation` | WAL truncation by LSN — `truncate_before(lsn)` drops older entries |
| `TestEndToEnd` | Full pipeline integration: write → CDC → index/view/FTS → update → delete → catch-up → rebuild → lag → introspect |

Additionally:
- **`TestPerformance`** — 10k-record smoke test asserting the system handles moderate scale with correct counts.
- **`TestScan`** — Prefix-based key scanning on the storage engine.

## Patterns

**1. Derived system as CDC consumer.** Every derived system (`SecondaryIndex`, `MaterializedView`, `FullTextSearch`) is registered via `db.add_derived_system()` and receives CDC events on `db.flush()`. This is the "unbundled" pattern: the database doesn't hardcode its secondary data structures — they're pluggable consumers of the change stream.

**2. Explicit flush.** Writes go to the WAL and storage immediately, but derived systems only update on `db.flush()`. This decouples write latency from downstream processing and makes lag observable — a deliberate design choice, not a test artifact.

**3. Catch-up semantics.** `add_derived_system(idx, catch_up=True)` replays the log so a newly-added consumer starts from a consistent snapshot. Without `catch_up=True`, the system only sees future events.

**4. State comparison for rebuild correctness.** `TestRebuild` captures `idx.get_state()` before and after `db.rebuild_system()`, asserting idempotence — live-processed state must equal replay-from-scratch state.

**5. Test isolation.** Every test creates a fresh `UnbundledDatabase()` instance. No shared state, no setup/teardown needed, no ordering dependencies between tests.

## Dependencies

**Imports:**
- `os`, `tempfile` — used only in `TestWAL.test_persistence` for file-based WAL testing
- `pytest` — test runner (used at `__main__` entry point, but no fixtures or parametrize)
- `unbundled_database` — the implementation module; imports 8 symbols covering the full public API

**Imported by:**
- Nothing — this is a leaf test file. It's consumed by `pytest` and the code-expert pipeline.

## Flow

The dominant pattern across tests:

1. **Create** `UnbundledDatabase()` (and optionally derived systems)
2. **Register** derived systems via `db.add_derived_system(system)`
3. **Write** data via `db.put(key, value)` — returns a `CDCEvent`
4. **Flush** via `db.flush()` — pushes pending CDC events to all subscribed derived systems
5. **Query** the derived systems directly (`idx.query()`, `mv.query()`, `fts.search()`)
6. **Assert** expected state

The `TestEndToEnd.test_full_pipeline` walks the complete lifecycle: write → query → update → delete → catch-up → rebuild → lag check → introspection → storage rebuild. This is the single test that exercises every subsystem in concert.

## Invariants

- **LSN monotonicity**: WAL entries get strictly increasing LSNs starting from 1 (`e1.lsn == 1`, `e2.lsn == 2`).
- **CDC event completeness**: Every `put` and `delete` returns a `CDCEvent` — put on new key → `insert` op, put on existing key → `update` op with `old_value`, delete on existing key → `delete` op with `old_value`.
- **Flush consistency**: After `flush()`, all derived system lag must be 0 — there can be no unprocessed events.
- **Rebuild idempotence**: `rebuild_system()` produces identical state to live processing (`live_state == rebuilt_state`).
- **Delete-on-nonexistent returns None**: `db.delete("nope")` returns `None`, not an error.
- **Truncation correctness**: `truncate_before(lsn)` removes entries with LSN < `lsn` and updates `earliest_lsn`.
- **Unsubscribe isolation**: Unsubscribing one CDC consumer doesn't affect others.

## Error Handling

The test file exercises **absence of errors** rather than error paths:
- Missing keys return `None` (not exceptions)
- Deleting nonexistent keys returns `None`
- Querying an empty index returns `[]`
- Querying a materialized view for an unknown group key returns `0`

There are no tests for invalid inputs (e.g., non-dict values, None keys), schema violations, or concurrent access failures. The contract is optimistic — the implementation assumes well-formed inputs.

## Topics to Explore

- [file] `unbundled-database/unbundled_database.py` — The implementation these tests specify; understanding how CDC subscription, flush, and catch-up are wired internally
- [function] `unbundled-database/unbundled_database.py:UnbundledDatabase.flush` — The critical path that bridges writes to derived system updates; defines the consistency boundary
- [file] `unbundled-database/test_unbundled_database.py` — The "other" test file; likely contains implementation-level tests or developer-authored edge cases vs. this specification suite
- [general] `ddia-unbundling-concept` — DDIA Chapter 12's argument for decomposing databases into logs + derived views; the conceptual foundation for this implementation
- [file] `unbundled-database/test_tester_validation.py` — Likely a meta-test validating that the tester test suite itself is well-formed or complete

## Beliefs

- `unbundled-db-flush-is-consistency-boundary` — Derived systems are only updated when `db.flush()` is called; writes to the WAL and storage happen immediately but CDC consumers are decoupled
- `unbundled-db-cdc-events-carry-old-value` — Every update and delete CDC event includes the previous value (`old_value`), enabling derived systems to undo prior state (e.g., decrement a count, remove from an index)
- `unbundled-db-rebuild-is-idempotent` — Rebuilding a derived system from the full log produces identical state to live incremental processing
- `unbundled-db-catch-up-replays-full-log` — A derived system added with `catch_up=True` receives synthetic CDC events for all existing data, bringing it to current state without requiring a separate snapshot mechanism
- `unbundled-db-wal-lsn-starts-at-one` — The WAL assigns LSNs starting from 1 (not 0), with `latest_lsn == 0` indicating an empty log

