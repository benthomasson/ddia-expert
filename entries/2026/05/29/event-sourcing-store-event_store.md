# File: event-sourcing-store/event_store.py

**Date:** 2026-05-29
**Time:** 07:58

# `event-sourcing-store/event_store.py`

## Purpose

This file implements an **event sourcing system** — one of the core patterns from DDIA Chapter 11 (Stream Processing). It owns three responsibilities:

1. **Durable, append-only event storage** — events are immutable facts, never updated or deleted
2. **Projections** — derived read models built by replaying events through registered handlers
3. **Snapshots** — periodic state checkpoints so projections don't have to replay from the beginning every time

This is the kind of infrastructure that backs systems like bank account ledgers or order histories, where you store *what happened* rather than *current state*, and derive current state on demand.

## Key Components

### `Event` (dataclass)

The fundamental unit of the system. Each event captures:
- `event_id` — a monotonically increasing global sequence number (1-based)
- `stream_id` — groups events into logical aggregates (e.g., `"order-123"`)
- `event_type` — a string tag like `"ItemAdded"` that projections dispatch on
- `data` — the event payload as a plain dict
- `timestamp` — wall-clock time of append
- `metadata` — optional sidecar dict for correlation IDs, user context, etc.

Events are *structurally* immutable (dataclass), though Python doesn't enforce this at runtime since `frozen=True` isn't set.

### `EventStore`

The append-only log. Its contract:

- **`append(stream_id, event_type, data, ...)`** — writes a single event. Supports optimistic concurrency via `expected_version`: if the stream's current version doesn't match, raises `ConcurrencyConflict`. Returns the created `Event`.
- **`append_batch(stream_id, events, ...)`** — atomically appends multiple events to one stream. The concurrency check happens once *before* any writes, so either all events land or none do (within a single process — there's no WAL or transaction log backing this).
- **`read_stream(stream_id, from_version)`** — returns events for a stream, filtered by `event_id >= from_version`. Note: `from_version` filters on global `event_id`, not stream-local position.
- **`read_all(from_position)`** — global ordered read, the foundation for projections' `catch_up`.
- **`stream_version(stream_id)`** — returns the count of events in a stream (used for optimistic concurrency checks).

Persistence is optional: pass `persist_path` and events are appended as newline-delimited JSON. On startup, the file is replayed to rebuild in-memory state.

### `Projection`

A pull-based read model. You register handlers via `when(event_type, handler)` where the handler signature is `(state: dict, event: Event) -> None` (mutates `state` in place). Calling `catch_up()` replays unprocessed events from the store.

Key details:
- `_position` tracks the last processed `event_id`, so `catch_up` is idempotent and incremental
- `_state` is a plain dict — the projection's handlers define its shape
- Snapshots are opt-in via `snapshot_interval` — every N events, state is deep-copied into `store._snapshots`

### `LiveProjection(Projection)`

A push-based variant that subscribes to the store's `_subscribers` list. Events are processed synchronously inside `append()`, so the projection is always up-to-date after every write. Same snapshot logic applies.

### `reconstruct_state(store, stream_id, handlers, up_to)`

A standalone utility that replays a single stream's events through handlers to produce state at a point in time. This is the classic event-sourcing "rehydrate an aggregate" operation — useful for ad-hoc queries or temporal debugging without maintaining a persistent projection.

## Patterns

- **Event Sourcing** (DDIA Ch. 11) — store facts, derive state. The `EventStore` is the system of record; projections are disposable and rebuildable.
- **CQRS-lite** — writes go through `append`, reads go through projections or `reconstruct_state`. The read and write models are separate, though they live in the same process.
- **Optimistic Concurrency Control** — `expected_version` implements OCC. No locks; the writer declares what version it thinks the stream is at, and the store rejects the write if someone else got there first.
- **Observer Pattern** — `_subscribers` is a simple pub/sub mechanism. `LiveProjection` uses it; you could also attach any `Callable[[Event], None]`.
- **Newline-Delimited JSON (NDJSON)** — the persistence format. Each line is a self-contained JSON record, making it easy to append without rewriting the file and to stream-process on recovery.

## Dependencies

**Imports**: All stdlib — `copy`, `json`, `os`, `dataclasses`, `datetime`, `typing`. No external dependencies.

**Imported by**: `test_event_store.py` and `test_verify.py` — the test suites exercise the store, projections, snapshots, and concurrency behavior.

## Flow

### Write path
1. Caller invokes `append()` or `append_batch()`
2. If `expected_version` is set, current stream version is checked → `ConcurrencyConflict` on mismatch
3. A new `Event` is created with the next sequential `event_id`
4. The event is appended to `_events` and its index is recorded in `_streams[stream_id]`
5. If persistence is enabled, the event is serialized to NDJSON and appended to disk
6. All `_subscribers` are notified synchronously

### Read path (Projection catch-up)
1. `catch_up()` calls `read_all(from_position=self._position + 1)`
2. For each event, if a handler is registered for its `event_type`, the handler mutates `_state`
3. `_position` advances to the event's `event_id`
4. If a snapshot interval is configured and the threshold is reached, `save_snapshot()` deep-copies state into `store._snapshots`

### Recovery path
1. `EventStore.__init__` detects an existing file at `persist_path`
2. `_load_from_file` reads each NDJSON line, reconstructs `Event` objects, and rebuilds `_events` and `_streams`
3. Projections can then `load_snapshot()` (if snapshots were persisted — currently they're only in-memory on the store) and `catch_up()` from there

## Invariants

- **Event IDs are globally sequential**: `event_id = len(self._events) + 1`, so IDs are 1-based and gap-free within a process lifetime.
- **Streams are append-only**: there is no delete or update operation on events.
- **Optimistic concurrency is version-count based**: `stream_version` returns the *count* of events in a stream, not the latest `event_id`. A stream with 3 events has version 3 regardless of their global event IDs.
- **Batch atomicity is process-local only**: `append_batch` does the concurrency check once, then appends all events. If the process crashes mid-batch, the NDJSON file could contain a partial batch — there's no write-ahead log or transaction boundary on disk.
- **Projection position tracks global event IDs, not stream versions**: `_position` is compared against `event_id`, which is global. This means `read_all(from_position=self._position + 1)` correctly resumes from where the projection left off across all streams.
- **Subscriber notification happens after persistence but inside the append call**: if a subscriber throws, subsequent subscribers won't be notified and the exception propagates to the caller — but the event is already persisted.

## Error Handling

- **`ConcurrencyConflict`** — the only custom exception. Raised by `append` and `append_batch` when `expected_version` doesn't match. This is the primary safety mechanism for concurrent writers.
- **File I/O errors** — not caught. If `_persist_event` or `_load_from_file` fails (disk full, corrupt JSON, permissions), the exception propagates directly to the caller.
- **Subscriber errors** — not caught. A failing subscriber in `_on_event` or in the `append` loop will short-circuit notification of remaining subscribers and bubble up.
- **No validation** on event data — any dict is accepted. Schema enforcement is left to the caller.

## Topics to Explore

- [file] `event-sourcing-store/test_event_store.py` — Covers the full API surface including concurrency conflicts, snapshots, and live projections
- [function] `event-sourcing-store/event_store.py:reconstruct_state` — The temporal query function; compare how it differs from projection-based reads for point-in-time state
- [general] `batch-atomicity-gap` — The NDJSON persistence has no transaction boundary for `append_batch`; worth comparing with the write-ahead-log implementation in `write-ahead-log/wal.py`
- [file] `change-data-capture/cdc.py` — CDC and event sourcing are complementary patterns from DDIA Ch. 11; compare how they solve the "derived data" problem differently
- [general] `snapshot-persistence` — Snapshots are stored in-memory on `store._snapshots` and lost on restart; a production system would persist them separately

## Beliefs

- `event-ids-are-1-based-sequential` — `event_id` starts at 1 and increments by 1 for each appended event, with no gaps within a process lifetime
- `stream-version-is-event-count` — `stream_version()` returns the number of events in a stream, not the latest event_id, and this is what `expected_version` checks against
- `batch-append-not-crash-safe` — `append_batch` can leave a partial batch on disk if the process crashes mid-write, since events are individually appended to the NDJSON file without a transaction marker
- `live-projection-is-synchronous` — `LiveProjection` processes events inside the `append()` call path via the subscriber mechanism, meaning the caller blocks until all projections have updated
- `snapshots-are-in-memory-only` — Projection snapshots are stored in `store._snapshots` (a dict attached at runtime), not persisted to disk, so they are lost on process restart

