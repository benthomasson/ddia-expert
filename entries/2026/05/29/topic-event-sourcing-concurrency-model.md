# Topic: How `expected_version` interacts with `append_batch` — can a batch fail partially, or is it atomic?

**Date:** 2026-05-29
**Time:** 11:13

## How `expected_version` Interacts with `append_batch`

### The Version Check: All-or-Nothing Gate

In `event_store.py:70-76`, `append_batch` checks `expected_version` **before** writing any events:

```python
def append_batch(self, stream_id: str, events: list[tuple[str, dict]],
                 expected_version: Optional[int] = None) -> list[Event]:
    if expected_version is not None:
        current = self.stream_version(stream_id)
        if current != expected_version:
            raise ConcurrencyConflict(...)
```

If the version doesn't match, `ConcurrencyConflict` is raised and **nothing is written** — the batch is rejected cleanly. This is the same pattern as the single `append` method at line 43-47, just applied to the batch entry point.

### The Atomicity Claim Is Overstated

The docstring says "Atomically append multiple events to a stream" (line 71), but the implementation doesn't fully deliver on that promise. Here's why:

**What works (the happy path):** The version check at the top acts as an optimistic concurrency guard. If it passes, the loop at lines 78-87 appends each event one by one to `_events` and `_streams`, and persists each to disk individually via `_persist_event`. Subscriber notifications are deferred until after all events are appended (lines 89-91), which is a genuine atomicity consideration — subscribers see a consistent batch, not partial state.

**What breaks (the failure path):** If `_persist_event` raises mid-batch (disk full, I/O error), some events will already be in `_events` and `_streams` in memory **and** written to disk, while later events in the batch won't be. There is no rollback mechanism — no `try/except` wrapping the loop, no snapshot of `_events` length to truncate back to. The grep for `lock|atomic|rollback|partial` returned zero matches, confirming there's no compensating logic anywhere in the file.

### Contrast with Single `append`

The single `append` method (line 38) has the same fragility — if `_persist_event` fails after the in-memory mutation, the store is inconsistent. But for a single event, the blast radius is smaller. With `append_batch`, a failure on event 2 of 5 leaves 2 events persisted and 3 lost, with the in-memory state reflecting 2 events that shouldn't exist without the other 3.

### Concurrency Model

This store is single-threaded (plain Python lists, no locks). The `expected_version` check works because there's no concurrent mutation between the check and the writes. In a multi-threaded or multi-process environment, the check-then-act pattern at lines 72-76 would be a classic TOCTOU race condition. The implementation assumes a single writer, which is reasonable for a reference implementation but worth noting.

### Summary

| Scenario | Behavior |
|---|---|
| Version mismatch | Clean rejection, nothing written |
| All events succeed | Fully consistent — events written, then subscribers notified |
| I/O failure mid-batch | **Partial write** — no rollback of in-memory or on-disk state |
| Concurrent writers | **Unsafe** — no locking around check-then-act |

The `expected_version` guard is solid as a pre-condition check, but the "atomic" claim in the docstring only holds when nothing goes wrong during the write loop.

## Topics to Explore

- [function] `event-sourcing-store/event_store.py:_persist_event` — Understanding the per-event disk write reveals why batch atomicity breaks under I/O failure
- [function] `event-sourcing-store/event_store.py:stream_version` — The version is derived from list length, not a stored counter — explore how this interacts with the concurrency model
- [general] `write-ahead-log-for-batches` — The WAL module in `write-ahead-log/wal.py` has its own `append_batch` — compare how it handles atomicity differently (it uses a log-structured approach)
- [function] `event-sourcing-store/event_store.py:LiveProjection` — Subscribers are notified after the batch loop; explore whether a `LiveProjection` could observe a partial batch if an exception occurs between lines 87 and 89
- [general] `batch-atomicity-gap` — Design what a true atomic batch would look like: write to a temp file, then atomically rename, or buffer in memory and flush once

## Beliefs

- `append-batch-version-check-is-precondition` — `append_batch` checks `expected_version` once before the write loop; it does not re-check after each event, so the guard is a pre-condition gate, not a per-event invariant
- `append-batch-no-rollback` — If an exception occurs mid-batch (e.g., disk I/O failure), already-appended events remain in `_events` and on disk with no rollback — the "atomic" docstring is aspirational, not enforced
- `append-batch-defers-subscriber-notification` — Subscriber callbacks are invoked only after all events in the batch are appended to `_events` and `_streams`, ensuring subscribers see the full batch or nothing (assuming no mid-loop exception)
- `event-store-single-writer-assumption` — The `expected_version` check-then-act pattern has no locking, making it safe only under a single-writer concurrency model

