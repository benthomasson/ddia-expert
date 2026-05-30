# File: unbundled-database/unbundled_database.py

**Date:** 2026-05-29
**Time:** 11:24

## Purpose

This file implements the **"database inside-out"** pattern from DDIA Chapter 12 — the idea that a modern data system is really a composition of independent subsystems (log, storage, indexes, views) glued together by a change stream, rather than a monolithic engine. Each component is explicit and replaceable: a write-ahead log captures intent, a storage engine materializes state, a CDC stream propagates changes, and derived systems (secondary indexes, materialized views, full-text search) consume those changes independently.

It serves as a teaching implementation: all the pieces you'd find spread across Kafka, PostgreSQL, and Elasticsearch are composed in ~350 lines of Python so you can see the wiring.

## Key Components

### Data Classes

- **`WALEntry`** — Immutable record of a mutation: an LSN (log sequence number), operation (`PUT`/`DELETE`), key, and optional value. This is the system-of-record representation.
- **`CDCEvent`** — Richer downstream representation: adds `old_value` for change tracking and uses semantic operation names (`insert`/`update`/`delete` instead of `PUT`/`DELETE`). The WAL doesn't distinguish insert from update — the CDC layer does, by checking whether `old_value` is `None`.

### Core Infrastructure

- **`WriteAheadLog`** — Append-only log with monotonically increasing LSNs. Supports optional file persistence (JSON-lines format). Key contract: `append()` assigns the next LSN atomically and returns the entry. `read_from(lsn)` and `truncate_before(lsn)` provide log consumption and garbage collection.

- **`StorageEngine`** — An in-memory key-value store that only mutates through `apply(WALEntry)`. It tracks its own `current_lsn` so you can tell how far it's caught up. The `rebuild(wal)` method clears state and replays the entire log — this is how crash recovery would work.

- **`CDCStream`** — The central pub-sub bus. Converts WAL entries into CDC events (adding old/new value semantics), stores the full event history, and fans out to subscribed `DerivedSystem` consumers. `process_pending()` pushes events to each consumer based on their individual position — each consumer tracks its own offset, so they can fall behind independently.

### Derived Systems (all extend `DerivedSystem`)

- **`SecondaryIndex`** — Inverted index mapping field values to primary keys. Maintains per-field `{value: set(keys)}` maps. On update, it removes old mappings before adding new ones.

- **`MaterializedView`** — Pre-computed aggregate (count or list) grouped by a field. Handles the add/remove bookkeeping for both insert/update and update/delete transitions.

- **`FullTextSearch`** — Word-level inverted index. Tokenizes by whitespace + lowercasing. Supports single-term and all-terms (intersection) queries.

### Facade

- **`UnbundledDatabase`** — Wires everything together. `put()` and `delete()` go through the full pipeline: WAL append → storage apply → CDC emit. `flush()` pushes CDC events to all derived systems. `add_derived_system()` can catch up a new consumer on existing data via `snapshot_and_stream`. `rebuild_system()` replays the full CDC history to reconstruct a derived system from scratch.

## Patterns

**Write-ahead logging**: Every mutation hits the WAL before the storage engine. The WAL is the source of truth; storage is a materialized view of the log.

**Event sourcing / CDC**: The `CDCStream` converts low-level log entries into semantic change events. This is the "derived data" pattern from DDIA — the primary store and all secondary structures are different representations of the same underlying log.

**Consumer position tracking**: Each `DerivedSystem` independently tracks its `position` (last processed LSN). `process_pending()` iterates the full event list and skips events the consumer has already seen. This is analogous to Kafka consumer group offsets.

**Snapshot-then-stream**: `CDCStream.snapshot_and_stream()` synthesizes insert events from the current storage state, then subscribes the consumer for future events — solving the "how does a new consumer catch up" problem without replaying the full log.

**Template method via ABC**: `DerivedSystem` defines the interface contract; concrete implementations provide `process_event()`, `rebuild()`, and `get_state()`.

## Dependencies

**Imports**: Only stdlib — `json`, `os`, `abc`, `dataclasses`, `typing`. No external dependencies.

**Imported by**: The three test files (`test_unbundled_database.py`, `tester_test_unbundled_database.py`, `test_tester_validation.py`) exercise the system end-to-end.

## Flow

A typical write flows:

1. `UnbundledDatabase.put(key, value)` captures the old value from storage
2. `WAL.append("PUT", key, value)` assigns an LSN and (optionally) persists to disk
3. `StorageEngine.apply(entry)` updates the in-memory store and advances its LSN
4. `CDCStream.emit(entry, old_value)` creates a `CDCEvent` (insert vs. update based on whether `old_value` existed) and appends it to the event log
5. The `CDCEvent` is returned to the caller — but derived systems haven't seen it yet
6. `UnbundledDatabase.flush()` → `CDCStream.process_pending()` pushes pending events to all subscribed consumers based on each consumer's position

This **lazy propagation** model means writes are fast (only WAL + storage), and derived system updates are batched. The caller controls when propagation happens by calling `flush()`.

## Invariants

- **LSN monotonicity**: `WriteAheadLog._next_lsn` only increments. Every entry has a strictly greater LSN than the previous one.
- **WAL-first mutation**: `StorageEngine` has no public mutation method other than `apply(WALEntry)` and `rebuild(wal)`. You cannot bypass the log.
- **Insert/update distinction lives in CDC, not WAL**: The WAL only knows `PUT`/`DELETE`. The CDC layer determines `insert` vs `update` by checking whether `old_value is None`.
- **Consumer position monotonicity**: Each `DerivedSystem._position` only advances (set to `event.lsn` after processing). Events are processed in LSN order.
- **Derived systems are rebuildable**: Every derived system implements `rebuild(events)` that clears state and replays from scratch, guaranteeing eventual convergence with the event log.

## Error Handling

Minimal — this is a teaching implementation. Notable behaviors:

- **`delete()` on nonexistent key** returns `None` without creating a WAL entry or CDC event. This is a silent no-op, not an error.
- **`SecondaryIndex._remove()`** uses `discard()` (not `remove()`) so removing a key that isn't indexed doesn't raise.
- **`MaterializedView`** clamps counts to zero with `max(0, ...)` to prevent negative counts from remove/add race conditions.
- **`FullTextSearch.search_all([])`** returns `[]` rather than raising on empty input.
- **File persistence** in `WriteAheadLog` does not handle corrupt JSON lines or partial writes — it assumes clean append-only I/O.
- **`snapshot_and_stream()`** directly sets `consumer._position` (accessing a "private" attribute), which is a coupling smell but avoids adding a public `set_position` to the ABC.

## Topics to Explore

- [file] `unbundled-database/test_unbundled_database.py` — See how the pipeline is exercised end-to-end: write → flush → query patterns, rebuild, lag monitoring
- [function] `unbundled-database/unbundled_database.py:CDCStream.snapshot_and_stream` — The catch-up mechanism for late-joining consumers; compare with Kafka's compacted topic approach
- [file] `change-data-capture/cdc.py` — The standalone CDC module in this repo; compare its event model and consumer protocol with the one embedded here
- [general] `derived-data-consistency` — Under what conditions can a derived system diverge from the primary store? Explore the gap between `put()` and `flush()`
- [function] `unbundled-database/unbundled_database.py:MaterializedView.process_event` — The most complex event handler; trace how update events with changed group-by values cause a decrement in the old group and increment in the new one

## Beliefs

- `wal-is-source-of-truth` — The StorageEngine has no direct mutation API; all state changes flow through WriteAheadLog.append → StorageEngine.apply, making the WAL the authoritative record
- `cdc-insert-update-distinction` — Insert vs. update is determined solely by whether CDCStream.emit receives a non-None old_value; the WAL itself only records PUT operations for both cases
- `lazy-derived-propagation` — Derived systems do not receive CDC events until UnbundledDatabase.flush() is explicitly called; writes to the primary store and propagation to derived systems are decoupled
- `consumer-independent-positions` — Each DerivedSystem tracks its own LSN position independently, allowing consumers to fall behind or catch up at different rates without blocking each other
- `derived-systems-rebuildable-from-cdc` — Every DerivedSystem can be reconstructed from scratch by calling rebuild() with the full CDC event history, guaranteeing convergence with the event log

