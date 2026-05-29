# File: event-sourcing-store/test_verify.py

**Date:** 2026-05-29
**Time:** 08:01

I can't access the implementation file, but the test file is self-documenting enough to give you a thorough explanation. Let me work from what's provided.

---

# `event-sourcing-store/test_verify.py`

## Purpose

This is a **specification verification script** — not a pytest test suite (no `test_` functions, no test class). It runs as a plain Python script (`python test_verify.py`) and exercises the `event_store` module's public API end-to-end, asserting that the "example usage from the spec" works correctly, as the docstring says. It serves as both an executable specification and an integration smoke test.

Its role in the project is to validate that the event sourcing implementation matches the expected behavior described in whatever spec guided the implementation. It's the equivalent of a "does the README example actually work?" check.

## Key Components

The file doesn't define reusable components — it's a linear script. But it exercises these imports from `event_store`:

| Component | Contract demonstrated |
|---|---|
| **`EventStore`** | Append-only log of events per stream. Supports `append()`, `read_stream()`, `read_all()`, `stream_version()`, `all_stream_ids()`, `append_batch()`. Optionally backed by a `.jsonl` file via `persist_path`. |
| **`Projection`** | Named read model that replays events through registered handlers (`when()`) to build state. Supports `catch_up()`, `save_snapshot()`, `load_snapshot()`. Tracks a `position` cursor and a `state` dict. |
| **`LiveProjection`** | A `Projection` variant that automatically updates when new events are appended — no manual `catch_up()` needed after the initial one. |
| **`reconstruct_state`** | Free function that replays a single stream's events through handlers up to a given version (`up_to=N`), returning the resulting state. Used for point-in-time reconstruction. |
| **`ConcurrencyConflict`** | Exception raised when `append()` is called with an `expected_version` that doesn't match the stream's current version — optimistic concurrency control. |

## Patterns

1. **Event sourcing**: Events are the source of truth. State is derived by replaying events through handler functions. The bank account example (open → deposit → withdraw) is the canonical DDIA illustration.

2. **Projections / Read models**: State is computed, not stored. `Projection` and `LiveProjection` both build a `state` dict by folding over events. Multiple projections can coexist over the same event store.

3. **Optimistic concurrency via `expected_version`**: The append with `expected_version=4` succeeds (the stream has 4 events), but `expected_version=3` correctly raises `ConcurrencyConflict`. This prevents lost-update anomalies in concurrent writers.

4. **Snapshot optimization**: `save_snapshot()` persists the projection's current state and position so a new `Projection` instance can `load_snapshot()` and resume from where it left off rather than replaying from event 0.

5. **Script-as-spec**: No test framework, just linear assertions with informative failure messages (`f"Got {value}"`). The final `print("All assertions passed!")` signals success. This pattern is common for verification scripts that should be simple to run in any environment.

## Dependencies

**Imports:**
- `event_store` (wildcard) — the entire public API: `EventStore`, `Projection`, `LiveProjection`, `reconstruct_state`, `ConcurrencyConflict`
- `tempfile`, `os` — only for the disk persistence test (creating and cleaning up a temp file)

**Imported by:** Nothing. This is a leaf script.

## Flow

The script runs **seven sequential scenarios**, each building on or independent of the previous:

1. **Core event append & read** (lines 4–9): Create a store, append 4 bank account events, verify `read_stream` returns them in order and the first event's type is correct.

2. **Projection catch-up** (lines 11–25): Register three event handlers on a `Projection`, call `catch_up()`, assert the balance is `0 + 100 - 30 + 50 = 120`.

3. **Point-in-time reconstruction** (lines 27–33): `reconstruct_state` with `up_to=2` replays only the first two events (AccountOpened + MoneyDeposited), asserting balance is `0 + 100 = 100`.

4. **Optimistic concurrency** (lines 35–41): Append with correct `expected_version=4` succeeds. Append with stale `expected_version=3` raises `ConcurrencyConflict`.

5. **Snapshot save/load** (lines 43–52): After catching up (balance now 130 after the +10 deposit), save a snapshot. A brand-new `Projection` loads that snapshot, gets the same state and position (5). Then a new event is appended and the new projection catches up to 110.

6. **LiveProjection** (lines 55–62): A `LiveProjection` catches up, then a new event is appended and the live projection's state updates automatically (135).

7. **Disk persistence** (lines 65–76): An `EventStore` with `persist_path` writes to a temp `.jsonl` file. A second `EventStore` opened with the same path reads the persisted events back.

8. **Batch append** (lines 79–85): `append_batch` writes three events atomically and returns the event objects.

9. **Stream isolation & global ordering** (lines 88–100): Events from different streams are isolated in `read_stream` but interleaved in insertion order in `read_all()`. Also verifies `all_stream_ids()`.

## Invariants

- **Stream ordering**: Events within a stream are ordered by version (1-indexed, sequential). `read_stream` returns them in that order.
- **Global ordering**: `read_all()` returns events in insertion order across all streams.
- **Version consistency**: `stream_version` equals the count of events in that stream. `expected_version` must match exactly or `ConcurrencyConflict` is raised.
- **Projection position tracking**: After `load_snapshot()`, `position` reflects how many events have been processed, so `catch_up()` only processes new events.
- **`up_to` is inclusive of the first N events**: `up_to=2` processes events at versions 1 and 2 (the first two events in the stream).
- **LiveProjection is reactive**: State updates without an explicit `catch_up()` after the initial one.

## Error Handling

Minimal, as befits a verification script:

- **`ConcurrencyConflict`**: The only exception tested. The script verifies it's raised on version mismatch and swallows it with `pass`. The `assert False` on the line before ensures the test fails if the exception is *not* raised.
- **Assertion messages**: Every `assert` includes an `f"Got {value}"` message to aid debugging if the expected value is wrong.
- **`try/finally` for cleanup**: The temp file for disk persistence is always deleted via `os.unlink(path)` in a `finally` block.

---

## Topics to Explore

- [file] `event-sourcing-store/event_store.py` — The implementation behind every API exercised here: how events are stored, how projections replay, how snapshots are serialized, and how `LiveProjection` subscribes to new events
- [function] `event-sourcing-store/event_store.py:reconstruct_state` — How point-in-time reconstruction works and whether `up_to` is version-based or index-based
- [file] `event-sourcing-store/test_event_store.py` — The pytest-style test suite, likely covering edge cases and failure modes this script doesn't
- [general] `event-sourcing-concurrency-model` — How `expected_version` interacts with `append_batch` — can a batch fail partially, or is it atomic?
- [general] `live-projection-subscription-mechanism` — Whether `LiveProjection` uses observer/callback registration or polling to stay current when new events arrive

## Beliefs

- `es-verify-is-script-not-pytest` — `test_verify.py` runs as a standalone Python script with inline assertions, not through a test framework
- `es-projection-position-tracks-global-offset` — `Projection.position` is a global event offset (value 5 after processing 5 events across all streams), not a per-stream version
- `es-expected-version-is-optimistic-lock` — `EventStore.append` with `expected_version` raises `ConcurrencyConflict` when the stream's current version doesn't match, implementing optimistic concurrency control
- `es-live-projection-auto-updates` — `LiveProjection` reflects new events immediately after `append()` without requiring a manual `catch_up()` call
- `es-snapshot-preserves-position-and-state` — `Projection.save_snapshot()` persists both `state` and `position`, enabling a new projection to `load_snapshot()` and resume incremental catch-up

