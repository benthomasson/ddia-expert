# Function: append_batch in event-sourcing-store/event_store.py

**Date:** 2026-05-29
**Time:** 11:11

## `EventStore.append_batch`

### Purpose

`append_batch` writes multiple events to a single stream in one logical operation. It exists to support domain operations that produce several events atomically — for example, a checkout that emits both `OrderPlaced` and `InventoryReserved`. The key difference from calling `append` in a loop: subscribers are notified *after* all events are stored, not interleaved with writes.

### Contract

**Preconditions:**
- `stream_id` is a non-empty string identifying the target stream.
- `events` is a list of `(event_type, data)` tuples. An empty list is technically valid (returns `[]`, notifies nobody).
- If `expected_version` is provided, the stream's current version must match it exactly.

**Postconditions:**
- All events in the batch are appended to `_events` and indexed in `_streams[stream_id]`, or none are (with one caveat — see below).
- Every subscriber is called once per event, in order, after all events are persisted.
- Returned events have contiguous, monotonically increasing `event_id` values.

**Invariant (intended but not enforced):** `event_id` values are globally unique and sequential (1-indexed). This holds only under single-threaded access.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `stream_id` | `str` | Logical stream to append to (e.g., `"order-123"`) |
| `events` | `list[tuple[str, dict]]` | Batch of `(event_type, data)` pairs |
| `expected_version` | `Optional[int]` | Optimistic concurrency guard. If set, the stream must have exactly this many events, or `ConcurrencyConflict` is raised. Defaults to `None` (no check). |

Edge cases: if `events` is empty, no version check side-effects occur (the check still runs if `expected_version` is set), and an empty list is returned.

### Return Value

`list[Event]` — the newly created `Event` objects in insertion order, with their assigned `event_id`, `timestamp`, and all fields populated. The caller gets back the same objects that are now in the store's internal list, so mutating them would corrupt store state (no defensive copy is made).

### Algorithm

1. **Optimistic concurrency check** — If `expected_version` is not `None`, read the stream's current version (count of events in that stream). If it doesn't match, raise `ConcurrencyConflict` immediately, before any mutation.

2. **Event creation loop** — For each `(event_type, data)` tuple:
   - Assign `event_id` as `len(self._events) + 1` (1-indexed global sequence).
   - Construct an `Event` dataclass instance with the current wall-clock time.
   - Append to the global `_events` list.
   - Update `_streams[stream_id]` with the index of the new event.
   - If disk persistence is enabled, write the event immediately via `_persist_event`.
   - Collect the event in `result`.

3. **Subscriber notification** — After all events are stored, iterate over `result` and call every subscriber with each event. This is the critical ordering distinction from `append`: subscribers see a consistent state where the entire batch is already in the store.

### Side Effects

- **In-memory mutation**: `_events` grows by `len(events)` entries; `_streams[stream_id]` gains corresponding index entries.
- **Disk I/O**: If `_persist_path` is set, each event is appended as a JSON line to the file *inside the loop*, not as a single atomic write. This means a crash mid-batch leaves a partial write on disk.
- **Subscriber callbacks**: Arbitrary external code runs for each event. A subscriber that raises will abort notification of remaining subscribers and remaining events — the store is already mutated at that point.

### Error Handling

| Exception | When | State after |
|-----------|------|-------------|
| `ConcurrencyConflict` | `expected_version` doesn't match current stream version | No mutation — raised before the loop |
| `KeyError` / `ValueError` | If an element in `events` isn't a 2-tuple | Partial mutation — events already appended before the bad tuple stay in the store |
| Subscriber exception | A subscriber raises during notification | All events are stored; only notification is incomplete |

There is no rollback mechanism. If the loop fails partway through (e.g., disk write error, bad tuple), events already appended are permanent. The "atomicity" in the docstring is logical (subscriber notification is deferred), not transactional.

### Usage Patterns

```python
store = EventStore()

# Append a batch with concurrency guard
store.append_batch("cart-42", [
    ("ItemAdded", {"sku": "A1", "qty": 1}),
    ("ItemAdded", {"sku": "B2", "qty": 3}),
], expected_version=0)

# Without concurrency check
store.append_batch("cart-42", [
    ("CartCheckedOut", {"total": 59.99}),
])
```

Callers typically use `expected_version` when replaying a command that must not conflict with concurrent writes. Without it, appends are unconditional.

### Dependencies

- `Event` dataclass — the store's event record type.
- `datetime.now()` — wall-clock timestamps with no timezone; not monotonic, not injected.
- `_persist_event` — JSON-lines file appender for durability.
- `_subscribers` — list of callbacks, populated externally (e.g., by `LiveProjection.__init__`).

### Assumptions Not Enforced by the Type System

1. **Single-threaded access** — `event_id = len(self._events) + 1` is a race condition under concurrent access. No locking exists.
2. **Mutable return** — returned `Event` objects are the same references stored internally. Callers must not mutate them.
3. **`data` is not copied** — the `dict` from each input tuple is stored directly in the `Event`. If the caller mutates it afterward, the stored event changes.
4. **Subscriber ordering** — subscribers see events in batch order, but nothing prevents a subscriber from appending more events to the store, which could interleave with the notification loop's view of state.
5. **Disk persistence is not atomic** — despite the docstring's "atomically," a crash mid-batch produces a partial file. Recovery via `_load_from_file` would load the partial batch with no indication it's incomplete.

## Topics to Explore

- [function] `event-sourcing-store/event_store.py:LiveProjection._on_event` — How real-time projections receive events from `append_batch`'s subscriber notification, and how the `event_id <= _position` guard prevents double-processing
- [function] `event-sourcing-store/event_store.py:append` — Compare with the single-event version to see the subscriber notification timing difference (inline vs. deferred)
- [general] `batch-atomicity-gap` — The gap between the docstring's "atomically" claim and the actual partial-failure behavior — what would true atomicity require?
- [file] `event-sourcing-store/test_event_store.py` — Test coverage for batch concurrency conflicts, empty batches, and subscriber interactions
- [general] `event-sourcing-snapshots` — How `Projection.save_snapshot` / `load_snapshot` interacts with batch-appended events during catch-up replay

## Beliefs

- `append-batch-defers-subscribers` — `append_batch` notifies subscribers only after all events in the batch are stored, unlike `append` which notifies immediately after each event.
- `append-batch-not-transactional` — Despite the "atomically" docstring, `append_batch` has no rollback: a failure mid-loop leaves already-appended events in the store and on disk.
- `event-id-is-global-sequence` — `event_id` is assigned as `len(self._events) + 1`, making it a 1-indexed global sequence number, not a per-stream version.
- `append-batch-shares-references` — Returned `Event` objects are the same instances stored in `_events`; no defensive copy is made for either the event or its `data` dict.
- `concurrency-check-is-pre-mutation` — The `expected_version` check runs before any state mutation, so a `ConcurrencyConflict` is safe and leaves the store unchanged.

