# File: unbundled-database/test_unbundled_database.py

**Date:** 2026-05-29
**Time:** 08:04

## Purpose

This is the primary test suite for the unbundled database implementation — a reference implementation of the "unbundling the database" concept from DDIA Chapter 12. The file validates that a write-ahead log, CDC stream, and multiple derived data systems (secondary indexes, materialized views, full-text search) stay consistent as data flows through the pipeline. It owns the behavioral contract for the entire `unbundled_database` module.

## Key Components

The file is organized as **16 numbered test classes**, each covering a distinct concern of the unbundled architecture. They progress from low-level primitives to full integration:

| Class | What it validates |
|---|---|
| `TestBasicPutGetDelete` | Core key-value CRUD semantics on `UnbundledDatabase` |
| `TestWAL` | `WriteAheadLog` — append, read-from-offset, LSN tracking, file persistence |
| `TestCDCEvents` | `CDCEvent` generation — correct `operation`, `old_value`, `new_value` per mutation type |
| `TestSecondaryIndex` | `SecondaryIndex` reacts correctly to inserts, updates, and deletes via CDC |
| `TestMaterializedView` | `MaterializedView` maintains aggregate state (`count`, `list`) across mutations |
| `TestFullTextSearch` | `FullTextSearch` — term indexing, multi-term AND queries, case insensitivity, delete cleanup |
| `TestCatchUp` | Late-joining derived systems replay existing CDC history to reach current state |
| `TestRebuild` | `rebuild_system` produces state identical to live incremental processing |
| `TestStorageRebuild` | `StorageEngine` can reconstruct itself entirely from the WAL |
| `TestLag` | Lag monitoring: positive after writes, zero after `flush()` |
| `TestPipelineIntrospection` | `get_pipeline_state()` returns accurate WAL size, record count, CDC event count, and per-system lag |
| `TestCDCOldValue` | Dedicated validation that `old_value` is `None` on insert, populated on update/delete |
| `TestDeleteCascade` | A single delete propagates correctly through all three derived system types simultaneously |
| `TestMultipleConsumers` | Independent CDC subscribers process events independently; unsubscribing one doesn't affect others |
| `TestLogTruncation` | WAL `truncate_before(lsn)` removes old entries and updates `earliest_lsn` |
| `TestEndToEnd` | Full integration — exercises the entire write → WAL → CDC → derived systems → catch-up → rebuild pipeline in sequence |
| `TestPerformance` | Smoke test at 10k records — verifies the system doesn't degrade at moderate scale |
| `TestScan` | `StorageEngine.scan(prefix)` filters keys by prefix |

## Patterns

**Two-phase write model.** Every test that checks derived systems follows the same pattern: `db.put()`/`db.delete()` to write, then `db.flush()` to push CDC events to consumers. This mirrors a real unbundled system where writes are durable immediately (WAL) but derived views update asynchronously. Tests that forget `flush()` would see stale derived state — this is intentional, not a bug.

**Fresh-instance-per-test.** Every test method creates its own `UnbundledDatabase()` — no shared fixtures, no `setUp`. This keeps tests fully isolated but means the test class groupings are purely organizational (no shared state within a class).

**Progressive complexity.** The numbered sections build from atoms (WAL, CDC events) to molecules (individual derived systems) to the full organism (end-to-end pipeline). A failure in section 2 (WAL) will cascade into failures in later sections, which aids root-cause diagnosis.

**Contract-via-return-value.** `db.put()` and `db.delete()` return `CDCEvent` objects directly, which tests inspect. This is a design choice that makes CDC a first-class part of the write API rather than a side-channel.

## Dependencies

**Imports:**
- `unbundled_database` — the entire module under test: `WALEntry`, `CDCEvent`, `WriteAheadLog`, `StorageEngine`, `CDCStream`, `SecondaryIndex`, `MaterializedView`, `FullTextSearch`, `UnbundledDatabase`
- `os`, `tempfile` — used only in `TestWAL.test_persistence` for file-backed WAL testing
- `pytest` — test framework (used at the bottom via `pytest.main`, but no fixtures or parametrize)

**Imported by:**
- `tester_test_unbundled_database.py` — likely a meta-test that validates these tests themselves
- `test_tester_validation.py` — appears to validate the tester layer

## Flow

The canonical data flow exercised across tests:

```
put(key, value)
  → WAL.append("PUT", key, value)     # durable write, assigns LSN
  → StorageEngine.put(key, value)     # in-memory primary store
  → CDCStream.publish(CDCEvent)       # event with operation/old/new
  
flush()
  → CDCStream delivers events to all subscribed derived systems
  → SecondaryIndex.process(event)     # updates inverted index
  → MaterializedView.process(event)   # updates aggregates
  → FullTextSearch.process(event)     # updates term index
```

For catch-up: `add_derived_system(system, catch_up=True)` replays the CDC event log to bring a new consumer to the current state without replaying the WAL.

For rebuild: `rebuild_system(name)` clears a derived system's state and replays from scratch, which must produce identical state to incremental processing (verified in `TestRebuild`).

## Invariants

1. **LSN monotonicity** — `wal.latest_lsn` strictly increases; `read_from(lsn)` returns entries from that LSN onward.
2. **CDC completeness** — every `put` produces exactly one `CDCEvent`; every `delete` of an existing key produces exactly one; deleting a nonexistent key returns `None`.
3. **Old-value tracking** — inserts have `old_value=None`; updates and deletes carry the previous value.
4. **Flush consistency** — derived systems reflect all mutations only after `flush()`; before flush, lag is positive.
5. **Rebuild equivalence** — `rebuild_system` produces state identical to live incremental processing (`live_state == rebuilt_state`).
6. **Subscriber independence** — unsubscribing one CDC consumer does not affect others.
7. **Truncation semantics** — `truncate_before(lsn)` removes entries with LSN < the given value and updates `earliest_lsn`.

## Error Handling

The tests validate a few negative/edge cases but don't test for exceptions:
- `get("nope")` returns `None` (not KeyError)
- `delete("nope")` returns `None` (not an error)
- No tests for malformed input, concurrent access, or persistence corruption

The test file itself does proper cleanup in `test_persistence` using `try/finally` with `os.unlink(path)` to remove temp files. Beyond that, error handling is minimal — this is a happy-path test suite that validates contracts, not resilience.

## Topics to Explore

- [file] `unbundled-database/unbundled_database.py` — The implementation being tested; understanding the `flush()` mechanism and CDC subscription model is essential
- [function] `unbundled-database/unbundled_database.py:UnbundledDatabase.rebuild_system` — How rebuild replays events and guarantees equivalence with live state
- [file] `unbundled-database/tester_test_unbundled_database.py` — The meta-test layer that validates these tests — understanding what it checks reveals additional contracts
- [general] `cdc-flush-semantics` — The two-phase write/flush model is the core architectural choice; explore how it maps to real async CDC systems (Debezium, Kafka Connect)
- [general] `ddia-ch12-unbundling` — DDIA Chapter 12's discussion of derived data systems and the "unbundled database" philosophy that motivates this entire implementation

## Beliefs

- `unbundled-db-put-returns-cdc-event` — `UnbundledDatabase.put()` and `delete()` return `CDCEvent` objects directly, making CDC a synchronous part of the write API
- `unbundled-db-flush-required-for-derived` — Derived systems (indexes, views, FTS) only see mutations after `flush()` is called; before flush, lag is nonzero
- `unbundled-db-rebuild-equals-live` — `rebuild_system()` must produce state identical to incremental live processing — this is a tested invariant, not just an aspiration
- `unbundled-db-catch-up-replays-history` — A derived system added with `catch_up=True` receives all historical CDC events, including the net effect of deletes
- `unbundled-db-wal-persistence-jsonl` — The WAL supports optional file-backed persistence via a `.jsonl` file path, and a new `WriteAheadLog` instance recovers entries from that file

