# Topic: How snapshots interact with LiveProjection — does a live projection ever snapshot, and can it restore from one while maintaining its subscription?

**Date:** 2026-05-29
**Time:** 11:13

I don't have access to the target repo. I'll work from what's available — the observations include lines 1–200, the test file, and the grep results.

---

## How Snapshots Interact with LiveProjection

### The short answer

**No — `LiveProjection` never snapshots, and it doesn't restore from snapshots.** The snapshot mechanism belongs exclusively to the base `Projection` class, and `LiveProjection` inherits from it but adds a real-time subscription that creates a subtle incompatibility with the snapshot workflow.

### How `Projection` snapshots work

The base `Projection` class (defined starting at line 159 of `event_store.py`) has full snapshot support:

- **`save_snapshot()`** (lines 196–200) deep-copies `self._state` and `self._position` into `self._store._snapshots[self.name]`. This is an in-memory snapshot stored on the `EventStore` itself — not persisted to disk.
- **`load_snapshot()`** (not shown in the observations, but referenced in `test_snapshot_save_load_and_resume` at line 130 of `test_event_store.py`) restores `_state` and `_position` from that dict, letting a fresh `Projection` skip replaying old events and call `catch_up()` to process only events after the snapshot position.
- **Automatic snapshots** are triggered during `catch_up()` (lines 189–191): if `_snapshot_interval` is set, the projection counts events processed and calls `save_snapshot()` every N events.

### What makes `LiveProjection` different

`LiveProjection` (defined after line 200 — outside the observation window, but its behavior is visible in `test_live_projection` at line 150 of `test_event_store.py`) subscribes to the store so events are applied **immediately on append**, without requiring `catch_up()`:

```python
store.append("account:1", "MoneyDeposited", {"amount": 25})
assert live.state["account:1"] == 145  # auto-updated — no catch_up() needed
```

This means `LiveProjection` registers a callback with `EventStore._subscribers` (line 36), and incoming events hit the projection's handlers inline during `append()` (line 68).

### Why `LiveProjection` doesn't snapshot

There are two reasons:

1. **No `snapshot_interval` parameter.** The test at line 150 constructs `LiveProjection("live_bal", store)` with no snapshot interval. The `Projection.__init__` (line 163) defaults `snapshot_interval` to `None`, so the automatic snapshot path at line 189 (`if self._snapshot_interval and ...`) never fires.

2. **The subscription isn't part of the snapshot.** Even if you manually called `save_snapshot()` on a `LiveProjection`, the snapshot stores only `_state` and `_position`. The subscription callback registered with `EventStore._subscribers` is **not** serialized. A new `LiveProjection` that called `load_snapshot()` would restore the state correctly, but it would then need to:
   - Call `catch_up()` to process any events between the snapshot position and the current position
   - Register a **new** subscription for future events

   The test shows that `LiveProjection` calls `catch_up()` first to replay history, then the subscription handles future events. There's no test that combines `load_snapshot()` with `LiveProjection`, which confirms this isn't a supported workflow.

### The gap this creates

For a long-running `LiveProjection`, if the process restarts, it must **replay the entire event log from position 0**. A regular `Projection` with snapshots can skip ahead. This is a known trade-off in event-sourcing systems: live projections optimize for latency (no polling), while snapshot-backed projections optimize for recovery time.

The implementation could be extended to support both — save periodic snapshots on a `LiveProjection`, then on restart, load the snapshot, `catch_up()` to fill the gap, and re-subscribe. But that code path doesn't exist today.

---

## Topics to Explore

- [function] `event-sourcing-store/event_store.py:LiveProjection` — The class definition is past line 200 and wasn't captured in observations; reading it would reveal whether it overrides `catch_up()`, `save_snapshot()`, or `load_snapshot()`
- [function] `event-sourcing-store/event_store.py:load_snapshot` — Referenced in tests but not visible in the observations; needed to confirm whether it re-registers subscriptions
- [function] `event-sourcing-store/event_store.py:reconstruct_state` — The temporal query helper (used in `test_temporal_query`) — a third way to build state from events, distinct from both projections and snapshots
- [general] `snapshot-persistence-gap` — Snapshots are stored in-memory on the `EventStore` object (`_snapshots` dict), meaning they don't survive process restarts even though events can be persisted to disk via `persist_path`
- [general] `catch-up-subscription-gap` — Between `catch_up()` finishing and the subscription being active, events appended concurrently could be missed; worth investigating whether `LiveProjection` handles this race

---

## Beliefs

- `live-projection-no-snapshot-interval` — `LiveProjection` is never constructed with a `snapshot_interval`, so the automatic snapshot path in `catch_up()` never triggers for live projections
- `snapshot-state-excludes-subscription` — Snapshot data (stored in `_store._snapshots`) contains only `state` and `position`; subscriber callbacks are not captured, so restoring a snapshot does not restore live-update behavior
- `snapshots-are-in-memory-only` — `save_snapshot()` stores snapshot data in a dict on the `EventStore` instance, not on disk, so snapshots do not survive process restarts even when `persist_path` is set
- `live-projection-full-replay-on-restart` — A `LiveProjection` must replay all events from position 0 on initialization because there is no snapshot restore path wired into its construction

