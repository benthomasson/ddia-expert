# Function: catch_up in event-sourcing-store/event_store.py

**Date:** 2026-05-29
**Time:** 11:14

## `Projection.catch_up` — Poll-based event replay

### Purpose

`catch_up` is the polling mechanism for a `Projection`. It reads all events the projection hasn't seen yet from the `EventStore`, applies registered handlers to update derived state, and optionally takes snapshots at configured intervals. This is the **pull-based** counterpart to `LiveProjection._on_event` (which is push-based via the subscriber callback).

It exists because projections are read models — materialized views derived from the event log. The event log is the source of truth; projections must periodically or on-demand replay new events to stay current.

### Contract

**Preconditions:**
- `self._store` is an `EventStore` with a functioning `read_all` method.
- `self._position` reflects the `event_id` of the last event this projection processed (0 if none).
- Handlers registered via `when()` must accept `(dict, Event)` and mutate the dict in place.

**Postconditions:**
- `self._position` equals the `event_id` of the last event in the store (if any new events existed), or is unchanged (if none).
- `self._state` reflects all handler mutations from newly processed events.
- `self._events_since_snapshot` is updated and potentially reset to 0 if a snapshot was triggered.

**Invariant:** `self._position` only advances forward, never backwards.

### Parameters

None beyond `self`. The method takes no arguments — it implicitly reads the gap between its own `_position` and the store's current tail.

### Return Value

Returns `int` — the count of events processed this call. Zero means the projection was already caught up. The caller can use this to decide whether derived state changed.

### Algorithm

1. **Query the gap**: Call `self._store.read_all(from_position=self._position + 1)` to get every event the projection hasn't seen. The `+1` skips the event already processed at the current position.

2. **For each event:**
   - **Dispatch**: If a handler is registered for `event.event_type`, call it with `(self._state, event)`. Events with no registered handler are **acknowledged but not handled** — the position still advances.
   - **Advance position**: Set `self._position = event.event_id`. This happens unconditionally, even for unhandled event types — the projection tracks global position, not just "interesting" events.
   - **Increment counters**: Bump `count` and `_events_since_snapshot`.
   - **Snapshot check**: If a `_snapshot_interval` is set and the events-since-snapshot counter has reached it, call `self.save_snapshot()` and reset the counter.

3. **Return** the total count.

### Side Effects

- **Mutates `self._state`**: Handlers modify the state dict in place. This is the whole point.
- **Mutates `self._position`**: Advances the high-water mark.
- **Mutates `self._events_since_snapshot`**: Tracks progress toward the next snapshot.
- **May call `save_snapshot()`**: Which deep-copies `self._state` and writes it to `self._store._snapshots` (a dict monkey-patched onto the store instance via `setattr` semantics in `save_snapshot`).
- **No I/O directly** — but `save_snapshot` mutates store state.

### Error Handling

None. If a handler raises, the exception propagates uncaught. This means a poison event (one whose handler throws) will halt the projection mid-catch-up. The position will have been advanced **up to but not including** the failing event's `event_id` assignment — actually, looking more carefully: `_position` is set *after* the handler call on the same iteration, so if the handler raises, `_position` is **not** advanced past the previous event. This means retrying `catch_up` will re-encounter the same poison event and fail again — a classic poison-message problem.

However, there's a subtlety: `_position` is updated **unconditionally** for events that have no handler. The position assignment is outside the `if` block. So if event N has a handler that raises, `_position` was set to event N-1's `event_id` on the prior iteration. On retry, `read_all(from_position=N)` will return event N again, and the handler will throw again. The projection is stuck.

### Usage Patterns

```python
# Typical: build a read model and catch up on demand
projection = Projection("order-totals", store)
projection.when("OrderPlaced", lambda state, e: ...)
projection.when("OrderCancelled", lambda state, e: ...)

# Initial catch-up from beginning
projection.catch_up()

# Later, after more events were appended
new_count = projection.catch_up()
if new_count > 0:
    print(f"Processed {new_count} new events")

# With snapshots for faster restart
projection = Projection("order-totals", store, snapshot_interval=100)
projection.load_snapshot()  # restore from last snapshot
projection.catch_up()       # only replay events since snapshot
```

The caller's obligation is to call `catch_up` whenever it wants fresh state. For automatic updates, use `LiveProjection` instead.

### Dependencies

- **`EventStore.read_all`**: The data source. Returns events filtered by `from_position`.
- **`self.save_snapshot`**: Writes snapshot to `self._store._snapshots`, which is an ad-hoc attribute on the store (not part of `EventStore.__init__`).
- **Registered handlers**: Arbitrary callables provided by the caller via `when()`.

### Assumptions Not Enforced by Types

1. **`event_id` values are monotonically increasing** — the `from_position=self._position + 1` logic assumes `event_id` acts as a sequential cursor. If IDs have gaps, events could be skipped.
2. **Handlers mutate `_state` in place** — the signature `Callable[[dict, Event], None]` doesn't enforce mutation; a handler that returns a new dict instead of mutating would silently do nothing.
3. **`read_all` returns events ordered by `event_id`** — if they came back out of order, position tracking would break (earlier events would be skipped on the next catch-up).
4. **No concurrent callers** — there's no locking. Two threads calling `catch_up` simultaneously would corrupt state.

---

## Topics to Explore

- [function] `event-sourcing-store/event_store.py:LiveProjection._on_event` — The push-based alternative; compare how it handles the same dispatch logic but with different consistency tradeoffs
- [function] `event-sourcing-store/event_store.py:Projection.save_snapshot` — How snapshot state is stored and the implications of monkey-patching `_snapshots` onto the store
- [general] `catch-up-subscription-gap` — The gap between the last snapshot position and current store position represents replay cost; explore how this scales
- [function] `event-sourcing-store/event_store.py:reconstruct_state` — A per-stream alternative to projections; compare event-replay scoping strategies
- [general] `poison-event-handling` — How production event-sourced systems handle handler failures without permanently stalling projections (dead-letter queues, skip-and-log)

## Beliefs

- `projection-catch-up-advances-position-for-unhandled-events` — `catch_up` advances `_position` for every event, not just those with registered handlers, so unhandled event types don't create replay gaps
- `projection-catch-up-poison-event-stalls` — If a handler raises during `catch_up`, the position is not advanced past the failing event, causing all subsequent `catch_up` calls to re-encounter and re-fail on the same event
- `projection-snapshot-interval-counts-all-events` — The snapshot interval counter increments for every event processed, including those without handlers, so snapshot frequency is based on total throughput not just handled events
- `projection-catch-up-assumes-sequential-ids` — The `from_position=self._position + 1` arithmetic assumes `event_id` values are contiguous integers with no gaps; the `EventStore.append` implementation satisfies this with `len(self._events) + 1`

