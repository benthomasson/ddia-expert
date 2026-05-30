# File: change-data-capture/cdc.py

**Date:** 2026-05-29
**Time:** 11:06

# `change-data-capture/cdc.py`

## Purpose

This file implements an in-memory Change Data Capture (CDC) system — a core pattern from DDIA Chapter 11 (Stream Processing). It owns the responsibility of intercepting every mutation to a simple relational database and recording it as an immutable, ordered event in an append-only log. Downstream consumers then derive secondary data structures (materialized views, search indexes) from that log rather than querying the primary database directly.

This is the "unbundled database" idea in miniature: writes go to one place, and all derived state is built asynchronously from the change stream.

## Key Components

### `ChangeEvent` (dataclass)
The unit of the log. Each event captures a before/after snapshot of a row, tagged with table name, primary key, operation type, a monotonic sequence number, and a wall-clock timestamp. The `before`/`after` fields follow the convention: INSERT has `before=None`, DELETE has `after=None`, UPDATE has both.

### `CDCLog`
The append-only event log. Internally a list with a monotonically increasing `_next_seq` counter. It provides:

- **`_append`** — package-private; only `CDCDatabase` should call it. Mints a new sequence number per event.
- **`read_from(position)`** — returns all events from a position onward. This is the primary consumer API — consumers track their own position and pull new events.
- **`read_range(start, end)`** — half-open range query `[start, end)`.
- **`compact()`** — log compaction: retains only the latest event per `(table, key)` pair, preserving relative order. Returns the count of events removed.
- **`current_position`** — the sequence number of the last appended event, or `-1` if the log is empty.

### `CDCDatabase`
A minimal in-memory relational database that emits change events on every mutation. Tables are defined with `create_table(name, columns, primary_key)` and stored as `{pk_value: row_dict}` maps. Every `insert`, `update`, and `delete` call appends a corresponding `ChangeEvent` to the shared `CDCLog`. Read operations (`select`, `scan`) do not generate events.

### `CDCConsumer`
A pull-based consumer that reads new events from the log. It maintains its own `_position` (cursor), supports handler registration filtered by table and/or operation type, and processes events in order via `poll()`. The `seek()` method allows rewinding or fast-forwarding the cursor.

### `MaterializedView`
A derived table that mirrors a source table's state by replaying the CDC stream. It supports an optional `transform` function that can reshape or filter rows — if the transform returns `None`, the row is excluded from the view (effectively a filter). Refreshed explicitly via `refresh()`.

### `SearchIndex`
A keyword inverted index maintained from the CDC stream. Tokenization is whitespace-split + lowercase on specified columns. On UPDATE, it fully removes old tokens before adding new ones — correct but not optimized for unchanged columns.

### `create_snapshot`
A standalone function that synthesizes the current database state as a list of synthetic INSERT events (all with `sequence_number=-1`) plus the current log position. This is the bootstrap mechanism: a new consumer can load the snapshot, then `seek()` to the returned position and start tailing the live log.

## Patterns

**Event Sourcing / Log-centric architecture**: The CDC log is the source of truth for all changes. Derived data structures are projections of that log, not independent copies.

**Pull-based consumption**: Consumers own their cursor position and explicitly call `poll()` or `refresh()`. There's no push/callback registration with the log itself — this avoids coupling and makes replay trivial.

**Before/after snapshots**: Every change event captures the full row state before and after the mutation. This makes events self-contained — a consumer doesn't need to query the source database to understand what changed.

**Log compaction**: `CDCLog.compact()` implements Kafka-style log compaction — keep only the latest event per key. This bounds log growth while preserving the ability to reconstruct current state.

**Snapshot + tail**: `create_snapshot` implements the standard pattern for bootstrapping new consumers without replaying the entire log from the beginning.

## Dependencies

**Imports**: Standard library only — `typing`, `enum`, `dataclass`, `datetime`. No external dependencies, which is consistent with the project's approach of implementing concepts from scratch.

**Imported by**: `test_cdc.py` and `tester_test_cdc.py` — the unit tests and a generated test wrapper, respectively.

## Flow

A typical lifecycle:

1. Create a `CDCDatabase`, define tables via `create_table`.
2. Perform mutations (`insert`, `update`, `delete`) — each appends a `ChangeEvent` to the internal `CDCLog`.
3. Create downstream consumers: a `CDCConsumer` with handlers, a `MaterializedView`, or a `SearchIndex`, all pointing to `db.cdc_log`.
4. Periodically call `poll()` / `refresh()` on each consumer — they read new events from their last position, process them, and advance their cursor.
5. Optionally call `cdc_log.compact()` to reclaim space.
6. To bootstrap a new consumer late, call `create_snapshot()` to get synthetic events + position, process the synthetics, then `seek()` to the position and start polling.

## Invariants

- **Monotonic sequence numbers**: `_next_seq` increments by 1 on every append. Sequence numbers are never reused, even after compaction. After compaction, the surviving events retain their original sequence numbers — they're not renumbered.
- **Primary key uniqueness**: `insert` raises `ValueError` on duplicate PK. `update` and `delete` raise `KeyError` on missing PK.
- **Before/after contract**: INSERT events always have `before=None`. DELETE events always have `after=None`. UPDATE events have both populated as full row copies.
- **Consumer position semantics**: `_position` is the *next* sequence number to read, not the last one read. After processing, it's set to `last_event.sequence_number + 1`.
- **Compaction preserves key coverage**: After `compact()`, there is exactly one event per `(table, key)` pair — the latest one by original list order.
- **Snapshot events use `sequence_number=-1`**: They are synthetic and not part of the real log. This sentinel distinguishes bootstrapped state from live events.

## Error Handling

Errors are raised eagerly at the database mutation layer:

- `KeyError` for referencing a nonexistent table or attempting to update/delete a missing row.
- `ValueError` for inserting a duplicate primary key.

There is no error handling at the consumer layer — if a handler raises, the exception propagates out of `poll()` / `refresh()`. There's no dead-letter queue, retry, or error event. Consumers that crash mid-poll will re-process all events from their last committed position on the next call (at-least-once semantics).

## Topics to Explore

- [file] `change-data-capture/test_cdc.py` — See how snapshot bootstrapping, compaction, and multi-consumer scenarios are exercised
- [function] `change-data-capture/cdc.py:CDCLog.compact` — The compaction walk-backwards algorithm and how it interacts with consumer positions that may point to removed events
- [general] `cdc-consumer-position-after-compaction` — What happens when a consumer's `_position` references a sequence number that was removed by compaction? The linear scan in `read_from` handles this correctly, but it's a subtle interaction worth tracing
- [file] `unbundled-database/unbundled_database.py` — The "unbundled database" module likely composes CDC with other derived stores, showing the full DDIA vision
- [general] `exactly-once-vs-at-least-once-delivery` — This implementation provides at-least-once semantics with no deduplication — worth comparing to how real CDC systems (Debezium, Kafka Connect) handle this

## Beliefs

- `cdc-log-sequence-numbers-survive-compaction` — `CDCLog.compact()` preserves the original sequence numbers of surviving events; it does not renumber them, so consumer positions remain valid after compaction
- `cdc-consumer-poll-is-at-least-once` — If a consumer crashes mid-`poll()`, the position is not advanced, so all events from that batch will be reprocessed on the next call
- `cdc-snapshot-uses-sentinel-sequence-number` — `create_snapshot` assigns `sequence_number=-1` to all synthetic events, distinguishing them from real log entries
- `cdc-materialized-view-transform-none-deletes` — When a `MaterializedView`'s transform function returns `None` for an INSERT or UPDATE event, the row is removed from the view, acting as a filter
- `cdc-search-index-full-reindex-on-update` — `SearchIndex` removes all old tokens and re-adds all new tokens on every UPDATE, even if only non-indexed columns changed

