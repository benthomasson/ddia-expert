# Topic: Consider whether there's a race between `catch_up()` finishing and the subscriber being registered — if an event is appended between those two steps, it could be missed

**Date:** 2026-05-29
**Time:** 11:12

The observations include the file up to line 200, but `LiveProjection` starts around line 210+. Let me work with what's available — the key lines are visible: line 225 (`store._subscribers.append(self._on_event)`) and line 227 (`def _on_event`).

---

## The Race Condition in `LiveProjection`

There is a real catch-up/subscribe race in the `LiveProjection` class, and understanding it requires tracing the exact sequence of operations.

### How `LiveProjection` works

`LiveProjection` inherits from `Projection` and does two things to stay up to date:

1. **Catch-up**: calls `catch_up()` (inherited from `Projection`, line 170) to replay all historical events up to the current global position.
2. **Subscribe**: registers `self._on_event` as a subscriber on the store (line 225: `store._subscribers.append(self._on_event)`).

The test at `test_event_store.py:148-155` shows the intended usage:

```python
live = LiveProjection("live_bal", store)
live.when(...)
live.catch_up()                              # step 1: replay history
store.append("account:1", "MoneyDeposited", {"amount": 25})
assert live.state["account:1"] == 145        # auto-updated via subscriber
```

### Where the race lives

The critical question is: **when does the subscriber get registered relative to `catch_up()`?**

Looking at line 225, the subscriber is registered in `LiveProjection.__init__()`, which runs **before** `catch_up()` is called. This means the actual sequence is:

1. `__init__` → subscriber registered (line 225)
2. User calls `catch_up()` → replays all historical events (line 170)
3. Future `append()` calls notify via `_on_event`

This ordering **avoids** the "event lost in the gap" problem — since the subscriber is registered first, any event appended after construction will be delivered via `_on_event`. But it introduces the **opposite** problem: **duplicate processing**.

### The duplicate processing window

If an event is appended **after** `__init__` (subscriber registered) but **before** `catch_up()` finishes replaying up to that point:

1. The subscriber fires immediately via `append()` → `for sub in self._subscribers: sub(event)` (line 64)
2. Then `catch_up()` also processes that same event when it reads it from `read_all()` (line 171)

The event gets applied to `self._state` **twice**.

### Why it's masked in single-threaded Python

This codebase uses no threading or locks (confirmed: no matches for `lock|Lock|threading`). In CPython's single-threaded execution:

- `catch_up()` calls `self._store.read_all()` which returns a snapshot of `self._events` at that moment
- No `append()` can interleave mid-iteration because there's no concurrency
- The subscriber can only fire during an `append()` call, which can only happen when `catch_up()` isn't running

So in practice, the race **cannot manifest** in the current single-threaded design. But the code has **no structural protection** against it — no lock, no position check in `_on_event`, no deduplication. If this were adapted for async or threaded use (e.g., a web server appending events while a projection catches up), the duplicate-processing bug would surface immediately.

### What a correct implementation would look like

A robust `LiveProjection` would either:

- **Subscribe first, then catch up, but deduplicate**: track `_position` in `_on_event` and skip events with `event_id <= self._position`
- **Catch up first, then subscribe with gap fill**: after subscribing, read any events between the catch-up position and the first subscriber-delivered event
- **Hold a lock**: serialize `catch_up` and `_on_event` so they can't overlap

The first approach is simplest and is the standard pattern in event-sourced systems.

---

## Topics to Explore

- [function] `event-sourcing-store/event_store.py:LiveProjection` — Read the full `__init__` and `_on_event` to confirm whether any position-based deduplication exists
- [function] `event-sourcing-store/event_store.py:append_batch` — Batch append notifies subscribers after all events are stored (line 93-95), creating a different timing window than single `append`
- [general] `subscriber-idempotency` — Whether `_on_event` checks `self._position` before applying, which would be the standard fix for this race
- [file] `unbundled-database/unbundled_database.py` — The `add_derived_system` with `catch_up=True` (line 431) faces the same pattern — compare how it handles the gap
- [general] `projection-snapshot-resume-race` — After loading a snapshot and calling `catch_up()`, the same duplicate window exists if the subscriber was registered in `__init__`

## Beliefs

- `live-projection-subscribes-before-catchup` — `LiveProjection.__init__` registers the subscriber (line 225) before the user calls `catch_up()`, making duplicate processing possible but event loss impossible in the current design
- `no-concurrency-control-in-event-store` — The event store uses no threading primitives (locks, mutexes, or thread-safe collections), relying entirely on single-threaded execution for correctness
- `subscriber-notification-is-synchronous` — `append()` notifies subscribers inline (line 64-65) within the same call, meaning the caller blocks until all subscribers have processed the event
- `catch-up-uses-event-id-filtering` — `catch_up()` calls `read_all(from_position=self._position + 1)` (line 171) to skip already-processed events, but `_on_event` has no equivalent guard

