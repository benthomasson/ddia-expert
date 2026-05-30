# Topic: DDIA Chapter 12's argument for decomposing databases into logs + derived views; the conceptual foundation for this implementation

**Date:** 2026-05-29
**Time:** 11:35

Now I have a thorough understanding of the codebase. Let me write the explanation.

---

# DDIA Chapter 12: Unbundling the Database into Logs + Derived Views

## The Argument

Chapter 12 of *Designing Data-Intensive Applications* makes a provocative claim: a traditional database is really a **bundle of loosely related subsystems** — storage engine, secondary indexes, materialized views, full-text search, replication — all wired together behind a single query interface. Kleppmann argues you can **unbundle** these components, connecting them through a shared, append-only log of changes rather than coupling them inside one monolithic process. He calls this "turning the database inside out."

The payoff: each derived system becomes independently rebuildable, replaceable, and scalable. Need a new search index? Attach a consumer to the log, replay from the beginning, and it catches up — no downtime, no migration. A materialized view got corrupted? Throw it away and rebuild from the log. The log is the source of truth; everything else is a cache.

This codebase implements that philosophy directly across four modules.

## The Architecture in Code

### 1. The Log as Ground Truth

The write path starts in `unbundled-database/unbundled_database.py`. The `WriteAheadLog` class (line 33) is an append-only sequence of `WALEntry` records (line 18), each carrying a monotonically increasing LSN, an operation (`PUT`/`DELETE`), a key, and an optional value.

The critical design choice: the `StorageEngine` (line 82) has **no direct mutation API**. Its only way to change state is through `apply(WALEntry)` (line 88). You can call `rebuild(wal)` (line 108) to throw the entire storage engine away and reconstruct it from the log. This makes the storage engine *derived data*, not primary data — exactly Kleppmann's point.

### 2. CDC as the Integration Backbone

Between the WAL and downstream consumers sits the `CDCStream` (line 89). It converts low-level WAL entries into semantically richer `CDCEvent` records (line 25) that carry both old and new values. The WAL only knows `PUT` and `DELETE`; the CDC layer determines `insert` vs. `update` by checking whether `old_value is None` (`unbundled-database/unbundled_database.py:102`).

Why `old_value` matters: `SecondaryIndex.process_event` (lines 152–155) must remove the stale index entry before adding the new one during an update. Without before-images, the index would accumulate phantom references — keys pointing to values they no longer hold.

The standalone `change-data-capture/cdc.py` module implements the same pattern in a relational setting. `CDCDatabase` (line 84) emits `ChangeEvent` records to a `CDCLog` (line 33) on every mutation. The `CDCConsumer` (line 140) is a pull-based reader that tracks its own cursor position and dispatches to registered handlers filtered by table and operation type. `CDCLog.compact()` (line 69) implements Kafka-style log compaction — keeping only the latest event per `(table, key)` — bounding log growth while preserving the ability to bootstrap new consumers.

### 3. Derived Systems as Pluggable Consumers

The `DerivedSystem` abstract base class (`unbundled_database.py:114`) defines the contract:

- **`process_event(event)`** — handle a single change incrementally
- **`rebuild(events)`** — clear internal state and replay from scratch
- **`position`** — track how far through the log this system has consumed

Three concrete implementations demonstrate the range:

| System | Location | What it derives |
|--------|----------|----------------|
| `SecondaryIndex` | line 131 | Inverted index: field values → primary keys |
| `MaterializedView` | line 189 | Pre-computed aggregates (count, list) grouped by field |
| `FullTextSearch` | line 241 | Word-level inverted index with whitespace tokenization |

All three consume the **same** CDC stream. Adding a new derived system requires zero changes to the write path — you call `db.add_derived_system(system)` and the next `flush()` feeds it pending events. This is unbundling in action.

### 4. The Two-Phase Write Model

The `UnbundledDatabase` facade (`unbundled_database.py:260`) exposes the full pipeline:

1. `put(key, value)` → captures old value → `WAL.append()` → `StorageEngine.apply()` → `CDCStream.emit()` — synchronous, fast
2. `flush()` → `CDCStream.process_pending()` → fans out events to all `DerivedSystem` consumers — explicit, batched

This lazy propagation means writes are fast (only WAL + storage), and the caller controls when derived systems catch up. The gap between `put()` and `flush()` is visible as **lag** — `get_lag(system_name)` returns `latest_lsn - system.position`. After `flush()`, lag drops to zero for all systems.

### 5. Event Sourcing: The Radical Variant

The `event-sourcing-store/event_store.py` module takes the log-centric idea further. In CDC, the database is primary and the log captures changes *to* it. In event sourcing, the event log **is** the database — there is no separate mutable store.

`EventStore` (line 29) is append-only. Events are organized into streams (e.g., `"order-123"`) with optimistic concurrency via `expected_version` (lines 43–47). The `Projection` class (line 159) is structurally identical to `DerivedSystem`: it tracks a `_position`, registers handlers by event type via `when()` (line 170), and catches up by calling `read_all(from_position=self._position + 1)` (line 174).

`LiveProjection` (line 219) adds a push-based variant — it subscribes to `EventStore._subscribers` and processes events synchronously inside `append()`, so the read model is immediately consistent.

The `reconstruct_state()` function (line 240) enables **temporal queries**: replay a stream's events through handlers up to a specific point in time. This is something event sourcing enables that CDC alone cannot — the full history survives, not just the latest state.

### 6. Stream Joins: Querying Across Derived Views

Once data is unbundled into separate streams, how do you query across them? The `stream-join-processor/stream_join_processor.py` module implements time-windowed joins (inner, left, full outer) between two event streams. It uses watermark-based expiration and deferred miss emission — an unmatched event only emits a miss when it expires past the time window, correctly handling late-arriving matches.

This is the query layer that completes the unbundled database. Without it, you can derive independent views but can't compose them.

## The Philosophical Arc

The four modules trace Chapter 12's argument as a progression:

1. **Start with the log** — `WriteAheadLog`, `EventStore`, `CDCLog` are all ordered, immutable records of what happened
2. **Derive everything else** — `StorageEngine`, `SecondaryIndex`, `MaterializedView`, `Projection` are each a different read-optimized view of the same truth
3. **Connect through CDC** — `CDCEvent`/`CDCStream`/`CDCConsumer` replace tight coupling with a stream contract
4. **Compose across streams** — `StreamJoinProcessor` provides the query layer for a world where data lives in multiple derived systems

The key operational insight: every derived system implements `rebuild()`. This means you can add, remove, or reconstruct any read-side component at any time, without downtime or schema migration. The log makes the system **operationally flexible** in a way a monolithic database is not.

---

## Topics to Explore

- [function] `unbundled-database/unbundled_database.py:StorageEngine.rebuild` — Proves the storage engine is truly derived: destroy it and reconstruct entirely from the WAL to see that the log, not the store, is the source of truth
- [function] `unbundled-database/unbundled_database.py:CDCStream.snapshot_and_stream` — The catch-up mechanism for late-joining consumers; compare with Kafka's compacted-topic bootstrapping approach
- [function] `event-sourcing-store/event_store.py:Projection.catch_up` — Compare with `DerivedSystem.process_event` to see how event sourcing and CDC converge on the identical position-tracking pattern
- [general] `flush-semantics-across-modules` — The unbundled database requires explicit `flush()` (lazy propagation) while event sourcing uses synchronous subscriber notification (`LiveProjection`) — explore the consistency/latency tradeoffs of push vs. pull delivery
- [file] `stream-join-processor/stream_join_processor.py` — The watermark and deferred-miss-emission logic implements the hard part of stream processing: deciding when an unmatched event will *never* match

---

## Beliefs

- `wal-is-source-of-truth` — `StorageEngine` has no direct mutation API; all state changes flow through `WriteAheadLog.append` → `StorageEngine.apply`, and `rebuild(wal)` reproduces identical state from the log alone
- `cdc-old-value-required-for-index-consistency` — `SecondaryIndex.process_event` depends on `CDCEvent.old_value` to remove stale index entries during updates; without before-images, incremental index maintenance produces phantom references
- `derived-systems-independently-position-tracked` — Each `DerivedSystem` tracks its own LSN `position` independently, allowing consumers to fall behind or catch up at different rates without coordinator state or blocking
- `lazy-propagation-via-explicit-flush` — Derived systems receive no CDC events until `UnbundledDatabase.flush()` is called; the gap between `put()` and `flush()` is observable as lag, providing predictable eventual consistency rather than strong consistency
- `event-sourcing-and-cdc-converge-on-projection-pattern` — Both `Projection.catch_up` and `DerivedSystem.process_event` track a position cursor, pull/receive ordered events, and apply type-dispatched handlers — the same structural pattern solving "derived data" from different starting points

