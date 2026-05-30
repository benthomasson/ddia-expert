# Function: LiveProjection in event-sourcing-store/event_store.py

**Date:** 2026-05-29
**Time:** 11:10

# `LiveProjection` — Auto-Updating Event Projection

## Purpose

`LiveProjection` is a subclass of `Projection` that subscribes to an `EventStore`'s event stream so its derived state updates **synchronously and automatically** whenever new events are appended. The base `Projection` class requires explicit `catch_up()` calls to process new events — `LiveProjection` eliminates that by hooking into the store's publish-subscribe mechanism.

This is the **push-based** counterpart to `Projection`'s **pull-based** model. It trades control over when processing happens for the guarantee that the projection is always current immediately after any `append()` or `append_batch()` call returns.

## Contract

**Preconditions:**
- The `store` must be a live `EventStore` instance (the projection registers a callback on it).
- Event handlers must be registered via `.when()` *before* events start flowing, or those early events will be silently skipped — there's no replay mechanism triggered by handler registration.

**Postconditions:**
- After construction, `self._on_event` is in `store._subscribers`.
- After any `store.append()` returns, this projection's `_state` and `_position` reflect that event (if a handler was registered for it).

**Invariants:**
- `_position` is monotonically increasing — the `event.event_id <= self._position` guard ensures idempotency.
- `_position` tracks the last *seen* event, not the last *handled* event. Events with no matching handler still advance the position.

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `str` | Identifier for this projection, used as the key in the snapshot store (`_store._snapshots[name]`). Must be unique per store if snapshots are used. |
| `store` | `EventStore` | The event store to subscribe to. The projection mutates `store._subscribers` and optionally `store._snapshots`. |
| `snapshot_interval` | `Optional[int]` | If set, auto-saves a snapshot every N events processed. `None` disables auto-snapshotting. A value of `0` would disable it too (falsy check). |

## Return Value

Constructor returns the `LiveProjection` instance. The class exposes state via inherited properties: `.state` (the derived dict) and `.position` (last processed event ID).

## Algorithm

**Construction:**
1. Calls `Projection.__init__` to set up `_handlers`, `_state`, `_position = 0`, and snapshot tracking.
2. Appends `self._on_event` to the store's `_subscribers` list — this is the subscription.

**On each event (`_on_event`):**
1. **Idempotency guard**: If `event.event_id <= self._position`, bail out. This prevents double-processing if `catch_up()` was called before subscribing, or if events somehow arrive out of order.
2. **Dispatch**: Look up `event.event_type` in `_handlers`. If a handler exists, call it with `(self._state, event)` — the handler mutates `_state` in place.
3. **Advance position**: Set `_position = event.event_id` regardless of whether a handler matched.
4. **Snapshot check**: Increment `_events_since_snapshot`. If we've hit the interval, save a snapshot and reset the counter.

## Side Effects

- **Mutates `store._subscribers`** at construction — adds a bound method reference. There is no unsubscribe mechanism; the projection stays subscribed for the lifetime of the store.
- **Mutates `self._state`** on every matching event — handlers modify the dict in place.
- **Mutates `store._snapshots`** when snapshot interval is reached — deep-copies state into the store.
- **Runs synchronously inside `store.append()`** — the callback executes within the append call stack. A slow handler blocks the append from returning. An exception in `_on_event` would propagate up through `append()` and leave the store in a partially-notified state (some subscribers called, others not).

## Error Handling

There is **none**. If a handler raises, the exception propagates up through `EventStore.append()` to the caller. The event is already persisted and appended to `_events` before subscribers are notified, so:

- The event is in the store (committed).
- The projection's `_position` was NOT updated (it crashes before line `self._position = event.event_id`).
- The projection is now behind — but the idempotency guard means a subsequent `catch_up()` call would reprocess the failed event successfully (assuming the handler was fixed or the data issue resolved).

This is actually a reasonable failure mode for event sourcing, but callers of `store.append()` need to be aware that subscriber exceptions surface there.

## Usage Patterns

```python
store = EventStore()
projection = LiveProjection("order_totals", store, snapshot_interval=100)

# Register handlers BEFORE any appends
projection.when("OrderPlaced", lambda state, e: state.update(
    {e.stream_id: state.get(e.stream_id, 0) + e.data["amount"]}
))

# Optionally catch up on historical events first
projection.load_snapshot()   # restore from last snapshot
projection.catch_up()        # replay events since snapshot

# From here, state is always current
store.append("order-1", "OrderPlaced", {"amount": 50})
assert projection.state["order-1"] == 50  # immediately visible
```

**Caller obligations:**
- Register handlers before events flow.
- If the store already has events, call `load_snapshot()` + `catch_up()` after handler registration to process the backlog — the live subscription only handles *new* events appended after construction.
- Handlers must not raise during normal operation (no error boundary).

## Dependencies

- **`Projection`** (base class) — provides `_handlers`, `_state`, `_position`, `catch_up()`, `save_snapshot()`, `load_snapshot()`.
- **`EventStore._subscribers`** — the pub-sub list. This is a plain `list[Callable]`, not a formal observer pattern.
- **`Event`** — the dataclass consumed by handlers. `event_id` is assumed to be monotonically increasing (assigned as `len(self._events) + 1` in the store).

---

## Topics to Explore

- [function] `event-sourcing-store/event_store.py:EventStore.append_batch` — Batch appends notify subscribers *after* all events are persisted, creating a subtle ordering difference from single appends
- [function] `event-sourcing-store/event_store.py:Projection.catch_up` — The pull-based counterpart; understanding both models clarifies when to use each
- [general] `subscriber-lifecycle-and-unsubscribe` — There's no way to remove a subscriber; explore what happens when a LiveProjection should be discarded
- [general] `snapshot-consistency-with-live-events` — What happens if `load_snapshot()` is called on a live projection that's already received events
- [file] `event-sourcing-store/test_event_store.py` — How the tests exercise the live projection, especially around ordering and catch-up interactions

## Beliefs

- `live-projection-synchronous-dispatch` — LiveProjection handlers execute synchronously inside `EventStore.append()`, blocking the caller until all subscribers complete
- `live-projection-no-unsubscribe` — Once constructed, a LiveProjection cannot be unsubscribed from its store; the callback reference persists in `_subscribers` indefinitely
- `live-projection-idempotent-by-position` — The `event_id <= _position` guard makes `_on_event` idempotent, allowing safe overlap between `catch_up()` and live subscription
- `live-projection-position-advances-without-handler` — Events with no registered handler still advance `_position`, preventing re-evaluation on catch-up
- `live-projection-no-error-boundary` — Handler exceptions propagate through `append()` to the caller, leaving the projection behind but recoverable via `catch_up()`

