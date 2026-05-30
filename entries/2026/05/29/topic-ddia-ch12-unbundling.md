# Topic: DDIA Chapter 12's discussion of derived data systems and the "unbundled database" philosophy that motivates this entire implementation

**Date:** 2026-05-29
**Time:** 11:27

I have enough from the observations to give a thorough explanation. Let me work with what's provided.

---

# The Unbundled Database: DDIA Chapter 12 in Code

## The Core Idea

Chapter 12 of *Designing Data-Intensive Applications* argues that a traditional database is really a *bundle* of loosely related subsystems: storage engine, secondary indexes, materialized views, full-text search, replication — all wired together behind a single query interface. Kleppmann's insight is that you can **unbundle** these components, connecting them through a shared log of changes rather than coupling them inside one monolithic process. He calls this the "database inside-out" pattern.

This codebase implements that philosophy directly. The `unbundled-database/` module is the clearest expression, but the idea motivates *every* module in the repository — `change-data-capture/`, `event-sourcing-store/`, and `stream-join-processor/` are all implementations of the same architectural decomposition at different levels of abstraction.

## How the Unbundled Database Works

### The Log as the Ground Truth

The architecture starts with a **write-ahead log** (`WriteAheadLog` in `unbundled-database/unbundled_database.py:33`). Every mutation — put or delete — is appended as a `WALEntry` (line 18) with a monotonically increasing LSN (log sequence number). This log is the single source of truth; the storage engine is just a derived view of it.

The `StorageEngine` (line 82) holds an in-memory key-value map, but it never mutates itself — it only applies WAL entries via `StorageEngine.apply()` (line 88). Its `rebuild()` method (line 108) demonstrates the key property: you can throw away the entire storage engine state and reconstruct it by replaying the WAL from the beginning. The storage engine is *derived data*, not primary data.

### CDC as the Integration Backbone

Between the WAL and the derived systems sits a **Change Data Capture stream**. The `CDCEvent` dataclass (line 25) captures not just what happened, but the *before* and *after* state for each change. This is essential — a secondary index needs to know the old value to remove the stale entry before adding the new one.

The test suite makes this explicit. In `test_unbundled_database.py:97-107`, an insert produces `operation == "insert"` with `old_value is None`, an update produces `operation == "update"` with both old and new values, and a delete produces `operation == "delete"` with `new_value is None`. This before/after contract is what allows derived systems to stay consistent without re-scanning the entire primary store.

### Derived Systems as Pluggable Consumers

The `DerivedSystem` abstract base class (line 114) defines the contract every derived view must satisfy:

- **`process_event(event: CDCEvent)`** — handle a single change incrementally
- **`rebuild(events: list[CDCEvent])`** — reconstruct from scratch by replaying history
- **`position`** — track how far through the log this system has consumed

Three concrete implementations demonstrate different use cases:

1. **`SecondaryIndex`** (line 131) — an inverted index mapping field values to primary keys. When processing an update, it first calls `_remove()` against the old value, then `_add()` against the new value (lines 152-155). This is why the CDC stream must carry `old_value`.

2. **`MaterializedView`** (line 189) — pre-computed aggregates (count, list) grouped by a field. The tests (`test_unbundled_database.py:148-185`) show it correctly handling inserts, updates that change the group key, and deletes — all driven purely from the CDC stream.

3. **`FullTextSearch`** — a text search index, referenced in the test imports (line 8 of the test file), providing yet another derived view from the same change stream.

The critical design point: all three consume the *same* CDC stream. Adding a new derived system doesn't require changing the write path. You call `db.add_derived_system(idx)` and the next `db.flush()` feeds pending events to all registered consumers. This is unbundling in action — the write path and the read-side indexes are decoupled through the log.

### The `rebuild()` Contract

Every `DerivedSystem` implements `rebuild()`, which clears its internal state and replays a complete event history. For `SecondaryIndex`, this is at line 174: reset the index, reset the position, and process every event sequentially. This means any derived system can be added *after* the database has been running — it just replays the full log to catch up. This is Kleppmann's argument for why logs make systems more operable: you can add new indexes, new views, new search engines without downtime or complex migration.

## The Same Pattern Across the Repository

### Change Data Capture (`change-data-capture/cdc.py`)

This module implements the pattern in a more traditional relational setting. `CDCDatabase` (line 84) is a table-oriented store where every `insert()`, `update()`, and `delete()` appends a `ChangeEvent` to a `CDCLog`. The `CDCConsumer` (line 140) polls the log and dispatches to registered handlers filtered by table and operation type. The `MaterializedView` (line 183) maintains a live copy of a table by consuming the CDC stream — the exact same derived-data pattern, just against relational tables instead of a key-value store.

The `CDCLog.compact()` method (line 69) is notable: it keeps only the latest event per `(table, key)`, which is log compaction as described in DDIA. This is how you bound the log size while still allowing new consumers to bootstrap — they replay the compacted log to get current state without processing every historical mutation.

### Event Sourcing (`event-sourcing-store/event_store.py`)

Event sourcing takes the log-centric philosophy further: the event log isn't just an implementation detail — it *is* the data model. The `EventStore` (line 29) is append-only. Events are organized into streams (line 32: `_streams: dict[str, list[int]]`), and the store supports optimistic concurrency via `expected_version` checks in `append()` (lines 43-47).

The `Projection` class (line 159) is the event-sourcing equivalent of a `DerivedSystem`. It registers handlers for specific event types via `when()` (line 170), and its `catch_up()` method (line 174) processes events sequentially, maintaining derived `_state` and a `_position` cursor. The snapshot mechanism (lines 185-192) periodically captures the projection state so that catch-up doesn't have to replay the entire log from the beginning — an optimization Kleppmann discusses as essential for practical event-sourced systems.

### Stream Joins (`stream-join-processor/stream_join_processor.py`)

This module addresses a problem that arises once you've unbundled: how do you combine data from *multiple* derived streams? The `StreamJoinProcessor` implements time-windowed joins (inner, left, full outer) between two event streams, complete with watermark-based expiration and late-event handling. This is the stream processing layer that Kleppmann argues completes the unbundled database — it's how you do "queries" across the separate derived views.

## The Philosophical Arc

Taken together, these modules trace Chapter 12's argument:

1. **Start with the log** (WAL, EventStore, CDCLog) — an ordered, immutable record of what happened
2. **Derive everything else** (StorageEngine, SecondaryIndex, MaterializedView, Projection) — each a different read-optimized view of the same truth
3. **Connect through CDC** (CDCEvent, CDCStream, CDCConsumer) — the integration mechanism that replaces tight coupling
4. **Compose across streams** (StreamJoinProcessor) — the query layer for a world where data lives in multiple derived systems

The `UnbundledDatabase` class ties it together as the facade: writes go to the WAL, changes flow through CDC, and any number of derived systems consume the stream independently. Each piece can be rebuilt, replaced, or scaled independently — which is exactly the operational advantage Kleppmann argues for over monolithic databases.

---

## Topics to Explore

- [function] `unbundled-database/unbundled_database.py:StorageEngine.rebuild` — Demonstrates that the storage engine is truly derived data: it can be destroyed and reconstructed entirely from the WAL, proving the log is the source of truth
- [file] `stream-join-processor/stream_join_processor.py` — The expiration/watermark logic (lines 155-200) implements the hard part of stream processing: deciding when an unmatched event will *never* match and should emit a miss
- [function] `event-sourcing-store/event_store.py:Projection.catch_up` — Compare this with `DerivedSystem.process_event` in the unbundled module to see how event sourcing and CDC converge on the same catch-up-from-position pattern
- [general] `flush-semantics-across-modules` — The unbundled database requires explicit `db.flush()` to push CDC events to derived systems (visible in every test), while the event store uses synchronous subscriber notification — explore the tradeoffs between push and pull delivery
- [file] `change-data-capture/cdc.py` — The `CDCLog.compact()` method (line 69) implements log compaction; compare it with Kafka's compaction semantics and consider what guarantees it provides to new consumers bootstrapping from the compacted log

---

## Beliefs

- `wal-is-source-of-truth` — The `StorageEngine` is fully derivable from the `WriteAheadLog`; calling `rebuild()` replays the entire WAL and reproduces identical state, making the log the authoritative record
- `cdc-old-value-required-for-consistency` — `SecondaryIndex.process_event` depends on `CDCEvent.old_value` to remove stale index entries during updates and deletes; without before-images, incremental index maintenance would produce phantom references
- `derived-systems-are-position-tracked` — Every `DerivedSystem` tracks a `position` (LSN) representing how far it has consumed the CDC stream, enabling independent catch-up without coordinator state
- `derived-systems-rebuildable-from-log` — All derived systems implement `rebuild(events)` which clears internal state and replays from an event list, allowing new consumers to be added at any time without schema migration or backfill jobs
- `unbundled-db-write-path-decoupled-from-reads` — Adding a new `DerivedSystem` via `add_derived_system()` requires zero changes to the write path (`put`/`delete`); the CDC stream is the only integration point between writers and readers

