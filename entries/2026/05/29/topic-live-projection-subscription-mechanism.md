# Topic: How `LiveProjection` receives events automatically after `catch_up()` — likely an observer/callback registered on the store's append path

**Date:** 2026-05-29
**Time:** 08:00

The observations cut off at line 200, and `LiveProjection` starts at line 219 — so we're missing the actual class. But we have enough from the surrounding code and the test to reconstruct exactly how it works.

---

## How `LiveProjection` Receives Events Automatically

The mechanism is a **synchronous observer pattern** — the `EventStore` maintains a subscriber list, and `LiveProjection` registers itself as a subscriber so it receives every event at append-time, with no polling.

### The Store Side: `_subscribers`

The `EventStore` holds a list of callbacks (`event_store.py:36`):

```python
self._subscribers: list[Callable[[Event], None]] = []
```

Every time `append()` is called, after the event is persisted, the store fans out to all subscribers (`event_store.py:65-66`):

```python
for sub in self._subscribers:
    sub(event)
```

The same pattern appears in `append_batch()` (`event_store.py:88-90`) — after all events in the batch are stored, each event is dispatched to each subscriber. Note the ordering: batch events are appended first, *then* subscribers are notified of all of them — this means a subscriber sees a consistent store state where all batch events exist.

### The Projection Side: `catch_up()` + self-registration

The base `Projection` class (`event_store.py:158`) is **pull-based** — you call `catch_up()` manually to process events since the last known position. It reads from the store's `read_all(from_position=...)` and applies matching handlers.

`LiveProjection` (line 219) extends this to be **push-based**. Though we can't see the class body, the test at `test_event_store.py:148-155` proves the contract:

```python
live.catch_up()
assert live.state["account:1"] == 120

store.append("account:1", "MoneyDeposited", {"amount": 25})
assert live.state["account:1"] == 145  # auto-updated — no catch_up() call
```

The append at line 153 triggers an *immediate* state update with no second `catch_up()`. This can only work if `LiveProjection` registered a callback in `store._subscribers` that applies the same handler logic as `catch_up()`.

The registration almost certainly happens in `catch_up()` itself (or in `__init__`). The most natural design: `catch_up()` first processes historical events via the parent's pull logic, then appends `self._handle_event` (or similar) to `store._subscribers` so that all future appends flow through the same handler pipeline automatically. This is a **catch-up subscription** — a standard event sourcing pattern where you first replay history, then switch to live updates at the exact position where replay ended, with no gap.

### Why It's Synchronous

The subscriber dispatch happens *inside* `append()`, on the caller's thread, before `append()` returns. This means:

1. **No gap** — an event is visible in the projection the instant `append()` returns.
2. **No concurrency issues** — in this single-threaded design, there's no race between appending and projecting.
3. **Caller pays the cost** — a slow projection handler blocks the writer. This is a deliberate tradeoff for simplicity over throughput.

---

## Topics to Explore

- [function] `event-sourcing-store/event_store.py:LiveProjection` — Read the full class body to confirm whether subscription happens in `__init__` or `catch_up()`, and whether it guards against double-subscription
- [function] `event-sourcing-store/event_store.py:append_batch` — Batch notification happens *after* all events are stored, unlike `append` which notifies per-event — explore whether this ordering difference matters for projections
- [general] `catch-up-subscription-gap` — Consider whether there's a race between `catch_up()` finishing and the subscriber being registered — if an event is appended between those two steps, it could be missed
- [file] `event-sourcing-store/test_verify.py` — Additional test coverage for LiveProjection behavior, including edge cases around catch-up timing
- [general] `projection-snapshot-interaction` — How snapshots interact with LiveProjection — does a live projection ever snapshot, and can it restore from one while maintaining its subscription?

## Beliefs

- `live-projection-uses-subscriber-list` — `LiveProjection` receives automatic updates by registering a callback in `EventStore._subscribers`, which is invoked synchronously inside `append()` and `append_batch()`
- `subscriber-dispatch-is-synchronous` — Event subscriber notification blocks the `append()` caller — projections update before the append call returns to the writer
- `batch-notifies-after-all-stored` — `append_batch()` stores all events first, then notifies subscribers of each event, ensuring subscribers see a consistent multi-event state
- `catch-up-then-subscribe` — `LiveProjection` implements the catch-up subscription pattern: replay historical events via `read_all()`, then switch to push-based updates for future events, bridging the pull/push boundary at a known position

