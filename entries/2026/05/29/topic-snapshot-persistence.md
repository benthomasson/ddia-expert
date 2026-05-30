# Topic: Snapshots are stored in-memory on `store._snapshots` and lost on restart; a production system would persist them separately

**Date:** 2026-05-29
**Time:** 11:07

## In-Memory Snapshots: No Persistence by Design

This observation touches two separate modules that both keep snapshot state purely in memory, but the `store._snapshots` pattern in the event sourcing store is the most illustrative.

### The event sourcing snapshots

In `event-sourcing-store/event_store.py`, snapshots are attached to the store object dynamically at lines 195–197:

```python
if not hasattr(self._store, '_snapshots'):
    self._store._snapshots = {}
self._store._snapshots[self.name] = {
    ...
}
```

This is a plain Python dict monkey-patched onto the store instance. There's no file I/O, no serialization, no `save`/`load`/`dump` method anywhere — the grep for `(persist|serialize|save|dump|load|restore)` returned zero matches across the entire project. When the process exits, that dict is gone.

The snapshot is retrieved at line 204 with a defensive `getattr` that defaults to an empty dict, meaning the code already expects snapshots to be absent — it silently falls back to replaying from the event log.

### The MVCC database has the same property

In `snapshot-isolation/mvcc_database.py`, the `MVCCDatabase.__init__` (lines 51–57) stores all state in plain dicts and sets:

```python
self._versions = {}          # key -> list[Version]
self._transactions = {}      # tx_id -> Transaction
self._committed = set()
self._aborted = set()
self._commit_timestamps = {}
```

Each transaction's snapshot is captured at `begin_transaction` (lines 70–72) by recording which transactions were active at that moment into `tx.active_at_start`. This is the core of MVCC snapshot isolation — but it's all in-process memory.

### Why this matters

Snapshots in the event sourcing store are an **optimization**, not a source of truth. The event log is the source of truth; a snapshot caches the aggregate's current state so you don't have to replay every event from the beginning. Losing snapshots on restart means the first read after restart replays the full event history — correct but slow.

For the MVCC database, losing state on restart means losing *all* data, not just a cache. A production MVCC system (PostgreSQL, for example) persists versions to a write-ahead log and data files, and records transaction status in a persistent commit log (pg_xact). This implementation skips all of that because it's a teaching tool focused on the isolation algorithm, not the storage engine.

### The production gap

A real system would need:
- **Event store snapshots**: periodically serialized to disk (or a separate table) keyed by aggregate name and version number, so replay starts from the last snapshot instead of event zero
- **MVCC state**: a WAL for versions, a persistent commit log for transaction status, and crash-recovery logic to rebuild in-flight transaction state

---

## Topics to Explore

- [file] `event-sourcing-store/event_store.py` — See how snapshots are created and consumed in the event sourcing aggregate lifecycle
- [function] `snapshot-isolation/mvcc_database.py:garbage_collect` — GC removes old versions that no active transaction needs; understand what "reachable" means without persistence
- [general] `wal-and-crash-recovery` — How a production MVCC system like PostgreSQL persists transaction state and recovers after a crash
- [function] `snapshot-isolation/mvcc_database.py:_is_visible` — The visibility rules that make snapshot isolation work, and why they depend on in-memory sets (`_committed`, `_aborted`, `active_at_start`)
- [general] `event-sourcing-snapshot-strategies` — Interval-based vs. event-count-based snapshotting policies and their trade-offs in real event-sourced systems

## Beliefs

- `snapshots-are-in-memory-dicts` — Event sourcing snapshots on `store._snapshots` are plain Python dicts with no serialization or disk persistence
- `snapshot-loss-triggers-full-replay` — When `_snapshots` is missing (line 204's `getattr` default), the system falls back to replaying all events, which is correct but slower
- `mvcc-state-entirely-volatile` — `MVCCDatabase` stores all versions, transaction status, and commit timestamps in-process memory with no WAL or persistence layer
- `no-persistence-api-exists` — A grep for `persist|serialize|save|dump|load|restore` across the project returns zero matches, confirming persistence is intentionally out of scope

