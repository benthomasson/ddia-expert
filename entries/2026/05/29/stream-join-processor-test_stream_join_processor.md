# File: stream-join-processor/test_stream_join_processor.py

**Date:** 2026-05-29
**Time:** 14:17

I can't read the implementation file due to permissions, but I have the full test file content. Here's the explanation based on what the tests reveal about the system.

---

# `stream-join-processor/test_stream_join_processor.py`

## Purpose

This is the test suite for a **stream join processor** — a windowed join engine that pairs events from two named streams by key within a time window. It implements the stream-stream join concept from DDIA Chapter 11 (stream processing), where you need to correlate events across streams (e.g., matching ad impressions with clicks) using temporal windowing.

The test file serves as both a correctness suite and a **living spec**. The 14 tests cover the full API surface: join semantics (inner/left/full outer), windowing, late arrival handling, buffering, aggregation, and statistics.

## Key Components

### Helper

- **`make_event(stream, key, timestamp, value=None)`** — Factory that wraps `StreamEvent` construction with sensible defaults (`value` defaults to `{}`). Keeps tests concise.

### Classes Under Test (from imports)

| Class | Role |
|-------|------|
| `StreamEvent` | Immutable event with `stream_name`, `key`, `value`, `timestamp` |
| `TimeWindow` | Defines window duration (e.g., `TimeWindow(5.0)` = 5-second window) |
| `StreamJoinProcessor` | Core engine — buffers events from two streams, matches by key within window |
| `JoinType` | Enum: `INNER`, `LEFT`, `FULL_OUTER` |
| `JoinResult` | Output record with `key`, `left_event`, `right_event`, `timestamp` |
| `TumblingWindowAggregator` | Groups `JoinResult`s into fixed-size non-overlapping windows, applies a reduce function |

### `StreamJoinProcessor` API (inferred from tests)

- **`process_event(event) → list[JoinResult]`** — Ingest an event, immediately return any matches found against the opposite stream's buffer.
- **`advance_time(timestamp) → list[JoinResult]`** — Move the watermark forward, expire old events, and emit miss results for unmatched events (left/full outer joins).
- **`buffer_size → tuple[int, int]`** — Current (left, right) buffer sizes.
- **`stats`** — Statistics object with `left_events_processed`, `right_events_processed`, `matches_emitted`, `misses_emitted`, `late_events_dropped`.
- **`get_results()`** — Drain the internal results buffer.
- **Constructor kwargs**: `allowed_lateness` (float), `on_result` (callback).

## Patterns

1. **Eager match emission**: `process_event` returns matches immediately rather than batching them. This is a push-based stream processing pattern — results are available the instant a match is found, not deferred to a flush.

2. **Watermark-driven expiration**: `advance_time()` simulates the watermark advancing. Events expire when `watermark - event.timestamp > window.duration`. This is the standard watermark model from Dataflow/Flink.

3. **Callback injection** (test 11): The `on_result` parameter decouples result delivery from the return value, supporting both pull (`results = process_event(...)`) and push (`on_result=lambda r: ...`) consumption patterns.

4. **Test numbering with comments**: Each test has a numbered comment header describing the scenario being tested — acts as a specification index.

5. **Spec-driven integration test**: `test_spec_example` at the bottom uses domain-realistic data (ad impressions/clicks) rather than synthetic keys, exercising the full lifecycle: ingest → match → miss → stats.

## Dependencies

**Imports:**
- `pytest` — test framework (imported but no fixtures or parametrize used; likely present for future `pytest.raises` usage)
- `stream_join_processor` — the implementation module, importing all 6 public types

**Imported by:** Nothing — this is a leaf test file.

## Flow

The typical test follows this pattern:

1. **Construct** a `StreamJoinProcessor` with two stream names, a `TimeWindow`, and a `JoinType`
2. **Feed events** via `process_event()`, capturing the returned matches
3. **Advance time** via `advance_time()` to trigger expiration/miss emission
4. **Assert** on match/miss counts, field values, buffer sizes, or statistics

The join matching logic works by: when an event arrives on stream L, scan stream R's buffer for events with the same key where `|event_L.timestamp - event_R.timestamp| <= window.duration`. A single left event can produce multiple matches if multiple right events exist in the window (test 9).

## Invariants

1. **Window boundary is strict**: Events where `|t_left - t_right| > window.duration` never match, even with the same key (test 2, k3 scenario: `|100 - 106| = 6 > 5`).

2. **Left join completeness**: A left event that matches produces a normal result and does **not** also produce a miss on expiration (test 4 explicitly checks this).

3. **Full outer join symmetry**: Both unmatched left *and* unmatched right events produce miss results — left misses have `right_event=None`, right misses have `left_event=None` (test 5).

4. **Late event cutoff**: An event is dropped when `event.timestamp < watermark - allowed_lateness` (test 8: timestamp 107 < 110 - 2 = 108).

5. **Buffer boundedness**: After processing 1000 events with a 5-second window, buffer sizes stay under 10 per side (test 13). The processor must actively expire old events, not accumulate unboundedly.

6. **Tumbling window boundaries**: Windows are aligned to multiples of the window size starting from 0 — `[0, 10)`, `[10, 20)`, etc. (test 12).

## Error Handling

The tests don't exercise explicit error paths (no `pytest.raises`). The error handling model is **silent drop with statistics**: late events aren't rejected with exceptions, they're dropped and counted in `stats.late_events_dropped`. This is the standard stream processing approach — you can't throw exceptions in a continuous pipeline.

---

## Topics to Explore

- [file] `stream-join-processor/stream_join_processor.py` — The implementation: how buffers are indexed by key, how the watermark drives expiration, and how miss detection works for left/full outer joins
- [function] `stream-join-processor/stream_join_processor.py:StreamJoinProcessor.advance_time` — The expiration and miss-emission logic is the most complex part — it must scan for unmatched events, emit misses, and clean buffers atomically
- [function] `stream-join-processor/stream_join_processor.py:TumblingWindowAggregator.advance_time` — How window alignment and closure works, and whether it handles the edge case of events exactly on window boundaries
- [general] `stream-join-semantics` — DDIA Ch. 11 discusses stream-stream joins vs. stream-table joins vs. table-table joins; this module implements stream-stream, worth understanding where the others differ
- [file] `stream-join-processor/tester_test_stream_join_processor.py` — The "tester" variant likely validates these tests against an alternative implementation or stricter contracts

## Beliefs

- `stream-join-inner-requires-key-and-window-match` — Inner join only produces a result when both key equality and `|t_left - t_right| <= window_duration` hold
- `stream-join-left-miss-emitted-on-expiration` — Unmatched left events in a LEFT join emit a JoinResult with `right_event=None` only when `advance_time` expires them, not eagerly
- `stream-join-late-events-dropped-silently` — Events arriving after `watermark - allowed_lateness` are dropped and counted in `stats.late_events_dropped`, never raising exceptions
- `stream-join-process-event-returns-immediate-matches` — `process_event` returns matches synchronously on each call rather than batching to a flush boundary
- `stream-join-buffers-bounded-by-window` — The processor actively expires events so buffer sizes remain proportional to `window_duration × event_rate`, not total events processed

