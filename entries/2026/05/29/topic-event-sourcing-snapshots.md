# Topic: How snapshot-accelerated reconstruction would work: load snapshot, then replay only events after the snapshot position

**Date:** 2026-05-29
**Time:** 11:08

# Snapshot-Accelerated Reconstruction in the Event Store

## The Core Problem

In event sourcing, the canonical way to get current state is to replay *every* event from the beginning. For a stream with thousands of events, this is expensive. Snapshot-accelerated reconstruction solves this by periodically saving the derived state at a known position, then only replaying the events that came *after* that checkpoint.

## How It Works: Two-Phase Reconstruction

The mechanism lives in the `Projection` class (`event-sourcing-store/event_store.py`).

### Phase 1: Save a Snapshot

`save_snapshot()` (line 193) captures two things as a pair:

```python
self._store._snapshots[self.name] = {
    "state": copy.deepcopy(self._state),   # the materialized view
    "position": self._position,             # which event we're caught up to
}
```

The `copy.deepcopy` is critical — it creates an independent copy of the state dict so future mutations during `catch_up()` don't corrupt the snapshot. The `position` records the `event_id` of the last event folded into this state.

### Phase 2: Load Snapshot + Replay Tail

When a new `Projection` instance needs to reconstruct state, it calls `load_snapshot()` (line 202), which restores both `_state` and `_position` from the snapshot. Then `catch_up()` (line 172) does the key trick:

```python
events = self._store.read_all(from_position=self._position + 1)
```

Because `_position` was set to the snapshot's position (say, event 5), `read_all` starts at event 6 — skipping every event the snapshot already incorporates. The projection then folds only the tail of the event log into the already-initialized state.

### The Test That Demonstrates It

`test_snapshot_save_load_and_resume` (test file, line 124) walks through the full cycle:

1. **Seed 5 events** → balance reaches 130
2. **`proj.catch_up()`** → processes all 5, position = 5
3. **`proj.save_snapshot()`** → freezes `{state: {"account:1": 130}, position: 5}`
4. **Create a fresh `p2`**, call `p2.load_snapshot()` → state is 130, position is 5 *without replaying anything*
5. **Append event 6** (withdraw 20)
6. **`p2.catch_up()`** → replays *only* event 6, arriving at 110

The reconstruction cost dropped from 6 events to 1.

## Automatic Snapshotting

The `Projection` constructor accepts `snapshot_interval` (line 157). During `catch_up()`, a counter tracks events processed since the last snapshot (line 179–182):

```python
self._events_since_snapshot += 1
if self._snapshot_interval and self._events_since_snapshot >= self._snapshot_interval:
    self.save_snapshot()
    self._events_since_snapshot = 0
```

This keeps the tail length bounded — at most `snapshot_interval` events need replaying on reconstruction.

## Contrast with Full Replay

The `reconstruct_state` function (used in `test_temporal_query`, test line 107) is the non-accelerated path: it replays all events from the start up to a given `event_id`. This is the right approach for *temporal queries* ("what was the balance after event 2?") but too expensive for "what is the balance *now*" on a long-lived stream.

## A Subtle Design Choice

Snapshots are stored in-memory on the `EventStore` instance (`self._store._snapshots`), not persisted to disk. This means snapshots survive within a process lifetime but not across restarts — a restarted system replays from scratch once, takes a snapshot, and subsequent reconstructions are fast. A production system would persist snapshots alongside the event log.

---

## Topics to Explore

- [function] `event-sourcing-store/event_store.py:reconstruct_state` — The full-replay path used for temporal queries; compare its cost profile to snapshot-accelerated reconstruction
- [function] `event-sourcing-store/event_store.py:LiveProjection` — Subscribes to events in real-time rather than polling with `catch_up()`; a different reconstruction-avoidance strategy
- [general] `snapshot-persistence-gap` — Snapshots live only in memory; explore what disk persistence (e.g., a separate snapshot file or table) would require and how it interacts with event log compaction
- [file] `write-ahead-log/wal.py` — The WAL's `replay(after_seq=...)` uses an analogous pattern: checkpoint a position, then replay only the tail; compare the two recovery models
- [general] `snapshot-interval-tuning` — Trade-off between frequent snapshots (less replay, more I/O and storage) and rare snapshots (more replay, less overhead); how would you choose the interval?

## Beliefs

- `snapshot-captures-state-and-position` — A snapshot stores both the materialized state *and* the event position, forming an atomic pair that allows replay to resume from the correct offset
- `catch-up-uses-position-plus-one` — `Projection.catch_up()` reads events starting at `self._position + 1`, which means loading a snapshot sets the exact boundary for which events still need replaying
- `snapshots-are-in-memory-only` — Snapshots are stored in a dict on the `EventStore` instance (`_snapshots`), not persisted to disk, so they do not survive process restarts
- `deepcopy-prevents-snapshot-corruption` — `save_snapshot()` uses `copy.deepcopy` on the state dict to ensure subsequent event processing doesn't mutate the saved snapshot

