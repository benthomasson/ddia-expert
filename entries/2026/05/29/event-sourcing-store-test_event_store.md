# File: event-sourcing-store/test_event_store.py

**Date:** 2026-05-29
**Time:** 06:22

# `event-sourcing-store/test_event_store.py`

## Purpose

This is the test suite for an event sourcing store — a reference implementation of the event sourcing pattern from DDIA (Chapter 11, stream processing / turning the database inside out). It validates twelve distinct capabilities of the `EventStore` system, organized as a specification: append/read, stream isolation, optimistic concurrency, batch writes, projections, temporal queries, snapshots, live projections, disk persistence, and global reads.

The file doubles as executable documentation. Each test is numbered and labeled to match a feature spec, making it easy to trace requirements to coverage.

## Key Components

### Shared event handlers

```python
on_opened, on_deposited, on_withdrawn
```

Pure functions with signature `(state: dict, event) -> None` that mutate a state dictionary in place. They model a bank account aggregate: open sets balance from `initial_balance`, deposit adds, withdrawal subtracts. State is keyed by `event.stream_id`, so a single state dict can track multiple accounts.

### `HANDLERS` dict

Maps event type strings to handler functions. Used by `reconstruct_state()` for temporal queries — it's the "fold function" definition for replaying events into state.

### `make_projection(name, store)`

Factory that creates a `Projection`, registers all three handlers via `.when()`, and returns it. Avoids repeating the registration boilerplate across tests.

### `_seed_account(store)`

Appends four canonical events to `"account:1"`: open (balance 0), deposit 100, withdraw 30, deposit 50. Final balance: 120. This is the shared fixture — most tests start from this known state.

## Patterns

**Arrange-Act-Assert** — every test follows this strictly. `_seed_account` is the shared arrange step; assertions are always on concrete values (120, 130, etc.), not relative comparisons.

**Numbered specification tests** — comments like `# --- 1. Basic append and read ---` tie each test to a feature requirement. This is a lightweight spec-by-example approach.

**No fixtures/setup classes** — each test creates its own `EventStore()`, giving full isolation. The only shared state is the helper functions, which are stateless.

**Event sourcing domain model** — the bank account is the classic event sourcing example. The test implicitly validates the core ES invariant: current state = fold(initial, events).

## Dependencies

**Imports from `event_store`:**
- `EventStore` — the append-only event log
- `Projection` — catch-up projection that polls for new events
- `LiveProjection` — projection that auto-updates on append (likely via callback/subscription)
- `ConcurrencyConflict` — exception for optimistic concurrency violations
- `reconstruct_state` — replays events up to a position to rebuild past state

**External:** `pytest` for test running and `pytest.raises` for exception assertions. `tempfile` and `os` for the disk persistence test only.

**Nothing imports this file** — it's a leaf in the dependency graph.

## Flow

The tests exercise five distinct flows through the event store:

1. **Write path**: `append()` / `append_batch()` → events get sequential `event_id` values, `stream_version` increments, `global_position` increments.

2. **Read path**: `read_stream(stream_id)` returns only that stream's events; `read_all(from_position=N)` returns the global log filtered by position.

3. **Projection path**: `Projection.catch_up()` reads events from `self.position` forward, applies handlers, advances position, returns count processed. Subsequent calls only process new events (incremental).

4. **Snapshot path**: `proj.save_snapshot()` persists state + position → `proj.load_snapshot()` restores them → `catch_up()` resumes from the snapshot position.

5. **Live projection path**: `LiveProjection.catch_up()` does initial replay, then subsequent `append()` calls on the store automatically invoke the projection's handlers — no explicit `catch_up()` needed.

## Invariants

- **Sequential event IDs**: Events within a stream are numbered 1..N. `test_append_and_read` asserts `events[0].event_id == 1` and `events[-1].event_id == 4`.
- **Stream isolation**: Events appended to stream `"a"` never appear in `read_stream("b")`.
- **Optimistic concurrency**: `append(..., expected_version=V)` succeeds only if current stream version equals V. Stale versions raise `ConcurrencyConflict`.
- **Projection idempotency**: Calling `catch_up()` twice without new events processes zero events the second time (tested implicitly — first call processes 4, second processes 1 new event).
- **Temporal reconstruction**: `reconstruct_state(..., up_to=N)` replays exactly the first N events, producing the state as it was at that point.
- **Persistence round-trip**: Writing to a JSONL file and loading into a new `EventStore` reproduces the same stream contents and version.

## Error Handling

Minimal — this is a test file, so errors are the things being tested:

- `test_optimistic_concurrency` uses `pytest.raises(ConcurrencyConflict)` to verify the store rejects stale writes. This is the only explicit error path tested.
- `test_disk_persistence` uses a `try/finally` to clean up the temp file, preventing test pollution.
- No tests for invalid event types, missing fields, or corrupt persistence files — the suite focuses on happy-path correctness and the one critical failure mode (concurrency conflicts).

## Topics to Explore

- [file] `event-sourcing-store/event_store.py` — The implementation behind all five classes/functions imported here; understanding how `LiveProjection` auto-updates (callback vs. observer pattern) is key
- [function] `event-sourcing-store/event_store.py:reconstruct_state` — The temporal query engine; how it handles the `up_to` parameter and whether it short-circuits or reads then truncates
- [general] `live-projection-subscription-mechanism` — How `LiveProjection` receives events automatically after `catch_up()` — likely an observer/callback registered on the store's append path
- [file] `event-sourcing-store/test_verify.py` — The companion test file; likely contains verification or property-based tests for the same store
- [general] `snapshot-storage-format` — Where and how `save_snapshot`/`load_snapshot` persist projection state — in-memory dict on the store, or a separate file

## Beliefs

- `es-event-ids-are-stream-scoped` — Event IDs are sequential per-stream starting at 1, not globally unique identifiers
- `es-global-position-tracks-all-streams` — `global_position` increments across all streams and equals the total number of events appended to the store
- `es-live-projection-auto-updates` — `LiveProjection` automatically applies new events on append without requiring an explicit `catch_up()` call after the initial one
- `es-projection-catch-up-is-incremental` — `Projection.catch_up()` only processes events after its current `position`, returning the count of newly processed events
- `es-persistence-uses-jsonl` — Disk persistence uses a JSONL (newline-delimited JSON) file format, loaded in full on `EventStore` construction

