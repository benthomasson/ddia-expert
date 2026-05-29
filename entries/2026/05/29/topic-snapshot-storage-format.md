# Topic: Where and how `save_snapshot`/`load_snapshot` persist projection state — in-memory dict on the store, or a separate file

**Date:** 2026-05-29
**Time:** 08:01

I have enough from the observations to give a complete answer.

## How `save_snapshot` / `load_snapshot` Persist Projection State

**Short answer: purely in-memory, on a dict hung off the `EventStore` instance. No file I/O involved.**

### `save_snapshot` — writing the snapshot

At `event_store.py:193-199`, `save_snapshot` does this:

```python
def save_snapshot(self):
    if not hasattr(self._store, '_snapshots'):
        self._store._snapshots = {}
    self._store._snapshots[self.name] = {
        "state": copy.deepcopy(self._state),
        "position": self._position,
    }
```

It lazily creates a `_snapshots` dict **on the `EventStore` instance** (`self._store`), then stores a deep copy of the projection's current `_state` and its `_position` (the last event ID it processed). The dict is keyed by projection name, so each projection gets one snapshot slot.

Key detail: `_snapshots` is **not** a declared attribute of `EventStore` — it's monkey-patched onto the instance via `hasattr`/setattr at line 195-196. The `EventStore` class itself (lines 28-150) has no knowledge of snapshots at all. This means snapshots live entirely in process memory and are **not** covered by `EventStore._persist_event` or `_load_from_file`, which only handle the event log.

### `load_snapshot` — reading it back

At `event_store.py:202` (and from the grep output at line 204), `load_snapshot` reads from the same dict:

```python
def load_snapshot(self) -> bool:
    snapshots = getattr(self._store, '_snapshots', {})
```

It uses `getattr` with a default empty dict, restores `_state` and `_position` from the stored snapshot, and returns `True` if one was found. This is how a new `Projection` instance pointing at the same `EventStore` can resume from a prior checkpoint without replaying every event from the beginning — as long as the store object is still alive in memory.

### Automatic snapshotting

During `catch_up()` (line 174-182), if a `snapshot_interval` was configured, the projection auto-saves after every N events:

```python
self._events_since_snapshot += 1
if self._snapshot_interval and self._events_since_snapshot >= self._snapshot_interval:
    self.save_snapshot()
    self._events_since_snapshot = 0
```

### What this means in practice

- **Snapshots don't survive process restarts.** If you restart the Python process, `_snapshots` is gone, and every projection must replay from event 0 (or from wherever the event log starts on disk).
- **The event log itself *can* persist to disk** via `EventStore(persist_path=...)`, but snapshots are purely ephemeral.
- **Multiple projections share the same `_snapshots` dict** on one store, keyed by `self.name`. Two projections with the same name would clobber each other's snapshots.

This is a deliberate simplification — a production system would typically persist snapshots to disk or a database alongside the event log. Here the snapshot mechanism demonstrates the *concept* (skip replaying N events by restoring from a checkpoint) without adding file-format complexity.

## Topics to Explore

- [function] `event-sourcing-store/event_store.py:catch_up` — How projections replay events and when automatic snapshots trigger
- [file] `event-sourcing-store/test_verify.py` — Shows the save/load round-trip across two `Projection` instances sharing one store
- [function] `event-sourcing-store/event_store.py:_persist_event` — Contrast with snapshots: events persist to JSONL on disk, snapshots don't
- [general] `snapshot-disk-persistence` — What it would take to make snapshots survive restarts (serialize to a sidecar file, integrate with `_load_from_file`)
- [file] `event-sourcing-store/test_event_store.py` — Tests at line 128-132 exercise snapshot save/load and verify catch-up resumes from the snapshot position

## Beliefs

- `snapshots-are-in-memory-only` — Projection snapshots are stored in a dict on the `EventStore` instance and do not persist to disk; they are lost on process restart
- `snapshots-dict-is-monkey-patched` — The `_snapshots` attribute is not declared on `EventStore`; it is lazily created by `Projection.save_snapshot` via `hasattr`/setattr
- `snapshot-keyed-by-projection-name` — Each projection's snapshot is stored under `self.name` in `_store._snapshots`, so two projections with the same name on the same store will overwrite each other
- `save-snapshot-deep-copies-state` — `save_snapshot` uses `copy.deepcopy` to isolate the stored snapshot from subsequent mutations to `_state`
- `event-log-persists-but-snapshots-do-not` — `EventStore` persists events to a JSONL file via `_persist_event`, but has no mechanism to persist or reload snapshots from disk

