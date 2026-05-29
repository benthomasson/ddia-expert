# File: stream-join-processor/stream_join_processor.py

**Date:** 2026-05-29
**Time:** 09:06

# `stream_join_processor.py` — Time-Windowed Stream Join

## Purpose

This file implements a **stream join processor** — the core primitive for combining two real-time event streams by matching events on a shared key within a time window. It corresponds to the stream-stream join concept from DDIA Chapter 11, where Kleppmann discusses how stream processors join events from two input streams (e.g., matching ad impressions with clicks, or correlating user activity across services).

The module owns two responsibilities:
1. **Stream joining** — buffering events from two named streams and emitting joined results when matching keys arrive within the configured time window.
2. **Tumbling window aggregation** — grouping join results into fixed, non-overlapping time windows and applying an aggregation function when windows close.

## Key Components

### Data Classes

- **`StreamEvent`** — The input unit. Carries `stream_name` (which stream it belongs to), `key` (the join key), `value` (arbitrary payload), and `timestamp` (event time, not processing time).
- **`JoinResult`** — The output unit. Contains the join key, the left and right events (either may be `None` for outer join misses), and a `join_timestamp` set to the later of the two matched events' timestamps.
- **`JoinStats`** — Mutable counters for observability: events processed per side, matches emitted, misses emitted, late events dropped.
- **`_BufferedEvent`** — Internal wrapper adding a `uid` (monotonic ID) and a `matched` flag. The `matched` flag is the mechanism that controls whether an outer-join miss is emitted at expiration.

### `JoinType` Enum

Three join semantics:
- `INNER` — only emit when both sides match.
- `LEFT` — emit unmatched left events as misses (right side `None`).
- `FULL_OUTER` — emit unmatched events from either side as misses.

### `TimeWindow`

A simple duration-based window. `contains(t1, t2)` checks whether two timestamps are within `duration` seconds of each other (absolute difference). This is a **symmetric** check — it doesn't matter which event arrived first.

### `StreamJoinProcessor`

The central class. Key API:

| Method | Contract |
|--------|----------|
| `process_event(event)` | Buffer the event, probe the opposite buffer for matches, advance watermark, expire old events. Returns join results produced by this event. |
| `advance_time(timestamp)` | Manually advance the watermark without an event. Triggers expiration and miss emission. Returns any miss results. |
| `get_results()` | Drain and return the accumulated results list (destructive read). |
| `stats` | Read-only access to `JoinStats`. |
| `buffer_size` | Returns `(left_count, right_count)` tuple of currently buffered events. |

### `TumblingWindowAggregator`

A post-join aggregation stage. Groups `JoinResult` objects into non-overlapping windows by `(key, window_start)` and applies a user-supplied `aggregate_fn` when `advance_time` closes a window. Window boundaries are computed by floor-dividing the timestamp by window size — standard tumbling window semantics.

## Patterns

**Event-time processing with watermarks.** The processor tracks a watermark (the highest event timestamp seen) and uses it to decide when events are "expired" and when late events should be dropped. This is the same model as Apache Flink/Beam watermarks, simplified to a single-source max-timestamp heuristic.

**Probe-on-arrival join.** When an event arrives, it is buffered on its side and then probed against the opposite buffer. This is a **nested loop join** scoped to matching keys — efficient because the buffers are keyed by join key (`defaultdict(list)`), so only events with the same key are compared.

**Deferred miss emission.** Outer-join misses aren't emitted eagerly. Instead, each buffered event carries a `matched` flag. When the event expires (falls below the watermark minus window duration), if `matched` is still `False` and the join type allows it, a miss result is emitted. This correctly handles the case where a match might arrive late.

**Destructive drain on `get_results`.** The results list accumulates across calls to `process_event` and `advance_time`, but `get_results` swaps it out and resets to empty. This is a pull-based consumption model — the caller decides when to consume.

## Dependencies

**Imports:** Standard library only — `typing`, `enum`, `dataclasses`, `collections.defaultdict`. No external dependencies.

**Imported by:** The test file `test_stream_join_processor.py` and a tester validation file `tester_test_stream_join_processor.py`, both exercising the join processor's behavior.

## Flow

### Event Processing (`process_event`)

```
1. Identify side (left or right) → increment stats
2. Late-event check: if timestamp < watermark - allowed_lateness → drop, return []
3. Compute new watermark (max of current and event timestamp)
4. Wrap event in _BufferedEvent, append to appropriate buffer
5. Probe opposite buffer for same key:
   - For each candidate, check TimeWindow.contains()
   - On match: create JoinResult, mark both sides matched, emit
6. Advance watermark to new value
7. Call _expire_events() to clean old entries and emit misses
8. Return slice of results produced during this call
```

The "return slice" trick (capturing `results_before = len(self._results)` then returning `self._results[results_before:]`) avoids copying and lets each `process_event` call return only its own results while still accumulating into the shared list.

### Expiration (`_expire_events`)

```
1. Compute cutoff = watermark - window_duration
2. For each buffer (left, right):
   - For each key, partition events into expired vs remaining
   - For expired + unmatched events: emit miss if join type allows
   - Replace buffer entry with remaining, or delete if empty
```

### Tumbling Aggregation (`TumblingWindowAggregator`)

```
1. add(result) → floor-divide timestamp to window_start, append to (key, window_start) bucket
2. advance_time(t) → close all windows whose end (start + duration) ≤ current window_start
   - Apply aggregate_fn to each closed window's results
   - Return sorted list of (key, window_start, window_end, aggregated_value)
```

## Invariants

1. **Watermark monotonicity.** The watermark only advances forward — `max(self._watermark, event.timestamp)`. `advance_time` explicitly guards `timestamp <= self._watermark`.

2. **Once matched, always matched.** The `_BufferedEvent.matched` flag is only ever set from `False` to `True`, never reset. This means an event that matches at least once will never produce a miss.

3. **Match symmetry.** Both sides of a match are marked: `buffered.matched = True` and `other.matched = True`. A single event can match multiple events on the opposite side (one-to-many join).

4. **Expiration cutoff = watermark - window_duration.** Events expire when their timestamp is strictly less than this cutoff. The window check on arrival uses `<=` (inclusive), so an event at exactly the boundary can still match but will expire on the next watermark advance past it.

5. **Left join only emits left-side misses.** In `_expire_events`, `JoinType.LEFT` only triggers miss emission for left-buffer events (`is_left_buf` must be `True`). Right-side unmatched events are silently discarded.

6. **No right-only join.** The enum doesn't include `RIGHT`. To get right-join semantics, you'd swap the stream names.

## Error Handling

There is essentially none — this is an in-memory, single-threaded processor with no I/O. The design relies on:

- **Type contracts** — the caller provides well-formed `StreamEvent` objects.
- **Silent dropping** — late events are silently dropped with only a counter increment (`late_events_dropped`). No exceptions, no callbacks.
- **No validation** — there's no check that `event.stream_name` matches either `_left_stream` or `_right_stream`. An event with an unknown stream name would be treated as a right-side event (since `_is_left` returns `False`), silently corrupting the join.

## Topics to Explore

- [file] `stream-join-processor/test_stream_join_processor.py` — See how join semantics (inner vs. left vs. full outer), late events, and tumbling aggregation are exercised in tests
- [general] `watermark-vs-processing-time` — Compare this event-time watermark model with processing-time approaches; DDIA Ch. 11 discusses the tradeoffs around event time, triggers, and allowed lateness
- [function] `stream_join_processor.py:_expire_events` — The most complex method; trace how the cutoff interacts with one-to-many matches and outer-join miss emission to verify correctness at window boundaries
- [file] `change-data-capture/cdc.py` — Another stream-oriented module in this repo; compare how CDC events flow vs. how stream joins buffer and match
- [general] `sliding-vs-tumbling-windows` — This module implements tumbling windows but uses a sliding-style symmetric time check for join matching; explore how a true sliding window aggregator would differ

## Beliefs

- `stream-join-one-to-many` — A single event can match multiple events on the opposite side; the matched flag is set on all participants, so none produce misses
- `stream-join-miss-deferred` — Outer-join misses are only emitted at expiration time (when events fall below `watermark - window_duration`), never eagerly on arrival
- `stream-join-left-asymmetric` — `JoinType.LEFT` emits misses only for unmatched left-side events; unmatched right-side events are silently dropped without any miss emission
- `stream-join-no-stream-validation` — The processor does not validate that an event's `stream_name` matches either configured stream; an unknown stream name is silently treated as the right stream
- `tumbling-window-floor-division` — `TumblingWindowAggregator` assigns windows by floor-dividing the timestamp by window size, producing aligned non-overlapping boundaries regardless of when events arrive

