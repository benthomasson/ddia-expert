# Topic: How projections and state reconstruction fit into Command Query Responsibility Segregation, the pattern DDIA discusses alongside event sourcing

**Date:** 2026-05-29
**Time:** 11:09

No access to the target repo directly, but the observations contain enough to give a thorough explanation.

---

# Projections and State Reconstruction in CQRS

## The Core Idea

CQRS separates a system into two sides: a **write model** that accepts commands and produces events, and one or more **read models** (projections) that transform those events into query-friendly shapes. This codebase implements exactly that pattern, even though the term "CQRS" never appears in the source — the architecture *is* CQRS without naming itself.

The `EventStore` is the write side. It accepts commands via `append()` and `append_batch()` (`event_store.py:41–96`), enforcing invariants like optimistic concurrency (`expected_version`). It knows nothing about balances, totals, or any query-friendly shape — it just records what happened.

The `Projection`, `LiveProjection`, and `reconstruct_state` functions are the read side. They consume the event log and build derived state optimized for queries.

## Three Flavors of Read Model

This codebase gives you three distinct approaches to building read-side state, each with different consistency and performance tradeoffs:

### 1. Pull-based Projection (`Projection`, line 153)

```python
proj = Projection("bal", store)
proj.when("AccountOpened", on_opened)
proj.when("MoneyDeposited", on_deposited)
proj.catch_up()   # explicitly pull new events
```

The projection tracks its `_position` (line 169) — the last event ID it has processed. When you call `catch_up()` (line 175), it reads only events *after* that position via `store.read_all(from_position=self._position + 1)`. This is poll-based CQRS: the read model is **eventually consistent** with the write model, updated on demand.

The test at `test_event_store.py:91–103` demonstrates the key property: after initial `catch_up()` processes 4 events, a subsequent `catch_up()` only processes the 1 new event. The projection doesn't re-read the entire log each time.

### 2. Push-based Live Projection (`LiveProjection`, line 219)

`LiveProjection` extends `Projection` and subscribes to the `EventStore._subscribers` list (line 34). When an event is appended, the store iterates `self._subscribers` and calls each callback (`event_store.py:68–69`). The live projection's handler fires immediately, making the read model consistent *within the same process* without polling.

The test at `test_event_store.py:139–145` proves this: after calling `catch_up()` to process historical events, a new `append()` is immediately visible in `live.state` without another `catch_up()` call.

This is the in-process equivalent of what Kafka consumers or CDC subscribers do in production CQRS systems.

### 3. Temporal Query via `reconstruct_state`

```python
past = reconstruct_state(store, "account:1", HANDLERS, up_to=2)
```

This replays a stream's events only up to a specific event ID, giving you the state *as it was at a point in time*. The test at `test_event_store.py:108–114` shows: after event 2 (AccountOpened + MoneyDeposited), the balance is 100; after event 3 (minus the withdrawal), it's 70.

This is CQRS's killer feature over mutable-state databases: because the write side stores events rather than overwriting current state, you can construct read models for *any* point in history. You're not limited to "what is the balance now" — you can ask "what was the balance at 3pm last Tuesday."

## Snapshots: The Performance Bridge

Replaying thousands of events to answer one query is expensive. Snapshots (`save_snapshot` at line 199, `load_snapshot` tested at `test_event_store.py:118–134`) solve this by checkpointing the projection's `_state` and `_position` at a point in time. A new projection instance can load the snapshot, then `catch_up()` only from the snapshot position forward.

The `_snapshot_interval` parameter (line 162) automates this — after N events processed, the projection automatically snapshots. This is the same pattern DDIA describes for event sourcing: snapshots are an optimization for read-model reconstruction, not a replacement for the event log.

## CDC as a Generalized CQRS Pipe

The `change-data-capture/cdc.py` module implements the same CQRS pattern but from a traditional database rather than an event store. The `CDCDatabase` captures every mutation as a `ChangeEvent` in a `CDCLog`, and downstream `CDCConsumer` and `MaterializedView` classes process these events to build derived state.

The `MaterializedView` in `cdc.py:190` is structurally identical to a `Projection` — it tracks `_position`, calls `refresh()` to pull new events, and applies a transform. The difference is the source: CDC derives events from database mutations (INSERT/UPDATE/DELETE on rows), while event sourcing starts with domain events. But the read-model construction works the same way.

This connects to DDIA's argument that CQRS and event sourcing converge with CDC — they're all instances of "derive read-optimized state from an append-only log."

## The Unbundled Database: CQRS at the Architecture Level

The `unbundled-database/unbundled_database.py` takes this further. `SecondaryIndex` (line 141) and `MaterializedView` (line 202) are both `DerivedSystem` subclasses that consume `CDCEvent` streams. They implement the same `process_event` / `rebuild` interface, making them interchangeable read models.

The `StorageEngine.rebuild()` method (line 108) reconstructs primary state from the WAL, just as projections reconstruct read-model state from events. Every piece of state in the system — primary data, secondary indexes, materialized views — is derived from the same log. This is DDIA Chapter 12's "database inside-out" vision: the log is the real database, and everything else is a cached materialization.

## Topics to Explore

- [function] `event-sourcing-store/event_store.py:LiveProjection` — How the subscriber mechanism achieves push-based consistency, and what happens if a handler throws mid-event
- [file] `unbundled-database/unbundled_database.py` — The `DerivedSystem` abstraction that unifies secondary indexes and materialized views as interchangeable CDC consumers
- [general] `projection-consistency-gaps` — The current `Projection.catch_up()` has no error handling or idempotency guarantees — what happens if it crashes mid-batch, and how production systems solve this with consumer offsets
- [function] `change-data-capture/cdc.py:CDCLog.compact` — Log compaction keeps only the latest event per key, which is safe for current-state projections but destroys the temporal query capability that `reconstruct_state` depends on
- [general] `write-side-validation-in-cqrs` — The `expected_version` concurrency check is the only write-side invariant enforced here; explore how richer domain validation (e.g., "balance must not go negative") would fit before events are appended

## Beliefs

- `projection-is-stateless-replay` — A `Projection` can be fully reconstructed from the event log at any time; its `_state` is a cache of derived data, not a source of truth
- `live-projection-uses-subscriber-push` — `LiveProjection` achieves immediate read-model consistency by subscribing to `EventStore._subscribers`, bypassing the poll-based `catch_up()` cycle for new events
- `snapshot-captures-position-and-state` — Snapshots store both the projection's `_state` dict and its `_position`, enabling catch-up to resume from the snapshot point rather than replaying the full log
- `cdc-and-event-sourcing-share-projection-pattern` — `CDCConsumer`/`MaterializedView` in `cdc.py` and `Projection` in `event_store.py` implement the same structural pattern (track position, pull new events, apply handlers) against different event sources
- `reconstruct-state-enables-temporal-queries` — `reconstruct_state` replays events up to a given event ID, allowing point-in-time state recovery — a capability that depends on the full event history and is destroyed by log compaction

