# Function: reconstruct_state in event-sourcing-store/event_store.py

**Date:** 2026-05-29
**Time:** 07:59

# `reconstruct_state`

## Purpose

This is a standalone utility for **temporal state reconstruction** — the core operation in event sourcing. Given a stream of events and a set of handler functions, it replays events in order to rebuild what the aggregate's state looked like at a specific point in time.

It exists as a complement to the `Projection` class. Where `Projection` maintains a continuously-updated read model across *all* streams, `reconstruct_state` is a one-shot, single-stream fold. You'd use it when you need the state of one specific aggregate (e.g., one order, one account) without maintaining a long-lived projection.

## Contract

**Preconditions:**
- `store` must be a valid `EventStore` with the stream already populated.
- `handlers` keys must match the `event_type` strings used when events were appended.
- Each handler must mutate the `state` dict in place (not return a new one).

**Postconditions:**
- Returns a dict representing the accumulated state after replaying all matching events up to `up_to`.
- The store is not modified — this is a pure read operation.

**Invariants:**
- Events are applied in the order returned by `read_stream`, which is insertion order within the stream.
- If `up_to` is provided, no event with `event_id > up_to` is applied.

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `store` | `EventStore` | The event store to read from. |
| `stream_id` | `str` | Identifies which aggregate's event stream to replay. |
| `handlers` | `dict[str, Callable]` | Maps event type names to functions with signature `(state: dict, event: Event) -> None`. Each handler mutates `state` in place. |
| `up_to` | `Optional[int]` | If provided, stops replay once an event's `event_id` exceeds this value. This is a *global* event ID, not a stream-local version number. |

**Edge cases:**
- If `stream_id` doesn't exist, `read_stream` returns `[]` and the result is `{}`.
- If `handlers` is empty, no events are applied regardless of what's in the stream — returns `{}`.
- If `up_to` is `0`, no events are applied (all event IDs are ≥ 1).
- Events whose `event_type` has no matching handler are silently skipped.

## Return Value

Returns a `dict` — the accumulated state after replay. The shape is entirely determined by the handlers; the function itself imposes no structure. The caller must know what shape to expect based on the handlers it passed in.

Returns an empty `dict` when the stream is empty, the stream doesn't exist, or no handlers matched any events.

## Algorithm

1. Fetch all events for the stream via `store.read_stream(stream_id)`. Note: this calls `read_stream` with the default `from_version=0`, so it reads the entire stream history.
2. Initialize an empty `state` dict.
3. Iterate through events in order:
   - **Cutoff check**: If `up_to` is set and the current event's `event_id` exceeds it, `break` — stop processing entirely. This works because events are ordered by `event_id`.
   - **Dispatch**: If the event's type has a registered handler, call `handler(state, event)`. The handler mutates `state` in place.
4. Return the accumulated `state`.

## Side Effects

None on the store. The function is read-only with respect to the `EventStore`.

However, the handlers themselves may have side effects — the function doesn't enforce purity. The `state` dict is created locally and passed to handlers, so state mutation is contained to the return value.

## Error Handling

There is no error handling. If a handler raises, the exception propagates uncaught and the caller gets a partially-applied state (lost, since `state` is a local variable). There's no transactional rollback.

If `read_stream` fails (shouldn't under normal operation since it's an in-memory list lookup), that also propagates uncaught.

## Usage Patterns

Typical usage — reconstruct a single aggregate's current state:

```python
handlers = {
    "AccountOpened": lambda s, e: s.update({"balance": 0, "owner": e.data["owner"]}),
    "Deposited": lambda s, e: s.update({"balance": s["balance"] + e.data["amount"]}),
    "Withdrawn": lambda s, e: s.update({"balance": s["balance"] - e.data["amount"]}),
}
current = reconstruct_state(store, "account-123", handlers)
```

Time-travel query — see what the state was at a past point:

```python
past_state = reconstruct_state(store, "account-123", handlers, up_to=42)
```

**Caller obligations:**
- Handlers must mutate `state` in place. A handler that returns a new dict without modifying `state` will silently lose its changes.
- The caller must provide handlers for every event type it cares about. Unhandled event types are skipped without warning.

## Dependencies

- `EventStore` and its `read_stream` method — the only external dependency.
- `Event` dataclass — handlers receive these, so the caller implicitly depends on its shape (`event_id`, `event_type`, `data`, etc.).
- No I/O, no imports beyond what's in the module.

## Relationship to `Projection`

This function does the same fold operation as `Projection.catch_up`, but with key differences:

| | `reconstruct_state` | `Projection` |
|---|---|---|
| Scope | Single stream | All streams (global read) |
| Lifecycle | One-shot | Long-lived, incremental |
| Time travel | Built-in via `up_to` | Not supported |
| Snapshots | No | Yes |
| Live updates | No | Yes (`LiveProjection`) |

## Assumptions Not Enforced by the Type System

1. **Handlers must mutate, not return.** The signature is `Callable` with no return type constraint. A handler returning a new dict instead of mutating `state` would silently produce wrong results.
2. **`up_to` is a global event ID, not a stream-local version.** The parameter name doesn't make this clear, and `read_stream` uses `from_version` which is *also* a global event ID despite the name suggesting a stream-local version. This naming inconsistency could lead to bugs.
3. **Events in the stream are ordered by `event_id`.** The `break` on `up_to` assumes monotonically increasing IDs. This holds because `_streams` stores indices in append order, but it's not documented as a guarantee.
4. **Handler dict keys exactly match `event.event_type` strings.** No normalization, no case folding — mismatched casing silently skips the event.

---

## Topics to Explore

- [function] `event-sourcing-store/event_store.py:Projection.catch_up` — The incremental counterpart to `reconstruct_state`; compare how it tracks position and applies snapshots
- [function] `event-sourcing-store/event_store.py:EventStore.read_stream` — Understanding its `from_version` semantics (global ID, not stream-local) is critical since `reconstruct_state` depends on it
- [file] `event-sourcing-store/test_event_store.py` — See concrete handler implementations and how `reconstruct_state` is exercised in tests
- [general] `event-sourcing-snapshots` — How snapshot-accelerated reconstruction would work: load snapshot, then replay only events after the snapshot position
- [general] `cqrs-read-models` — How projections and state reconstruction fit into Command Query Responsibility Segregation, the pattern DDIA discusses alongside event sourcing

## Beliefs

- `reconstruct-state-is-pure-read` — `reconstruct_state` never modifies the `EventStore`; it is a read-only fold over a single stream's events
- `reconstruct-state-up-to-is-global-id` — The `up_to` parameter filters on global `event_id`, not stream-local version number, despite operating on a single stream
- `reconstruct-state-no-snapshot-support` — `reconstruct_state` always replays from the beginning of the stream; it does not leverage snapshots, unlike `Projection`
- `handlers-must-mutate-state-in-place` — Both `reconstruct_state` and `Projection` expect handler functions to mutate the `state` dict in place rather than returning a new value
- `unhandled-event-types-silently-skipped` — Events whose `event_type` has no entry in the handlers dict are skipped without warning in both `reconstruct_state` and `Projection.catch_up`

