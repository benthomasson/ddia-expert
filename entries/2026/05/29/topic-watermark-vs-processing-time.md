# Topic: Compare this event-time watermark model with processing-time approaches; DDIA Ch. 11 discusses the tradeoffs around event time, triggers, and allowed lateness

**Date:** 2026-05-29
**Time:** 14:19

# Event-Time Watermarks vs. Processing-Time: The Stream Join Processor

## How This Implementation Uses Event-Time Watermarks

The `StreamJoinProcessor` in `stream-join-processor/stream_join_processor.py` implements a **pure event-time model**. Every decision — buffering, matching, expiring, and dropping — is driven by timestamps embedded in the events themselves, not by when the system processes them.

The watermark is the central coordination mechanism. It's initialized to negative infinity (line 81) and advances monotonically as events arrive:

```python
new_watermark = max(self._watermark, event.timestamp)   # line 114
```

This single line captures the core assumption: **the watermark is the system's best guess at "how far along are we in event time."** It never goes backward. When the watermark advances, it triggers expiration of buffered events whose timestamps have fallen below the window cutoff:

```python
cutoff = self._watermark - self._window.duration         # line 166
```

Events older than this cutoff are removed from buffers. For outer joins (`LEFT` or `FULL_OUTER`), unmatched events emit miss results before expiring — this is how the system produces "no match found" outputs.

## The Three Levers: Watermarks, Lateness, and Triggers

DDIA Ch. 11 identifies three fundamental tensions in stream processing around event time. This implementation addresses each one explicitly.

### 1. Watermark = "How complete are we?"

The watermark answers: *can we safely say that no more events with timestamp < W will arrive?* In this implementation, the answer is conservative — the watermark is simply the max timestamp seen. A real distributed system (like Apache Flink or Google Dataflow) would compute watermarks from source metadata, acknowledging that the max-seen heuristic can be wrong when events arrive out of order across partitions.

The `advance_time()` method at line 155 is the escape hatch for this. It lets an external orchestrator push the watermark forward without receiving an actual event — essential for handling idle streams or end-of-input:

```python
def advance_time(self, timestamp: float) -> list[JoinResult]:
    if timestamp <= self._watermark:
        return []
    self._watermark = timestamp
    ...
```

The tests use this heavily. `test_left_join_miss` calls `p.advance_time(110.0)` to force expiration and miss emission — simulating a trigger that fires based on watermark progress rather than event arrival.

### 2. Allowed Lateness = "How long do we wait for stragglers?"

The `allowed_lateness` parameter (line 74) creates a grace period below the watermark. The late-event check at line 109:

```python
if event.timestamp < self._watermark - self._allowed_lateness:
    self._stats.late_events_dropped += 1
    return []
```

This is the tradeoff DDIA describes: wider lateness windows mean more complete results but higher memory costs (buffers must be kept longer) and higher latency before results are finalized. The test at line 82 (`test_out_of_order_within_lateness`) demonstrates the sweet spot — an event at t=103 arrives after the watermark has advanced to 105, but with `allowed_lateness=3.0`, the cutoff is 102, so the event is accepted and successfully joins.

Contrast this with `test_late_event_dropped` (line 96): `allowed_lateness=2.0`, watermark at 110, cutoff at 108, and an event at t=107 is dropped. The system has decided completeness isn't worth the cost of keeping that buffer state around.

### 3. Triggers = "When do we emit results?"

This implementation uses an **eager trigger model** for matches and a **watermark-driven trigger** for misses. When a new event arrives and matches something in the opposite buffer, the result is emitted immediately (lines 125–148). But miss detection — knowing that no match *will ever come* — requires the watermark to advance past the window boundary, which happens either through new event arrival or explicit `advance_time` calls.

## What a Processing-Time Approach Would Look Like

A processing-time system would replace all of this with wall-clock logic: "buffer events for 10 real-world seconds, then emit whatever matched." The `TimeWindow.contains()` check (line 53) would compare `time.time()` deltas instead of event timestamps. There would be no watermark, no allowed lateness, and no `advance_time`.

The advantage is simplicity — no out-of-order headaches, no watermark computation, no lateness decisions. The disadvantage is **non-determinism and incorrectness under load**. If the system slows down (GC pause, network delay, backpressure), events that belong together in event-time can land in different processing-time windows. A 2-second network hiccup could cause a click that happened 3 seconds after an impression to appear 13 seconds later in processing time, falling outside a 10-second window and producing a false miss.

Notice how the tests in `test_stream_join_processor.py` never call `time.time()`. Every timestamp is explicit: `make_event("L", "k1", 100.0)`. This is a direct benefit of the event-time model — **the join logic is fully deterministic and testable** regardless of how fast or slow the test machine runs. A processing-time implementation would need sleeps or clock mocking, and would still be flaky under CI load.

## The Buffer Memory Tradeoff

The event-time model requires buffering, and `test_buffer_cleanup` (line 158) validates that this buffering is bounded. After processing 1000 events with a 5-second window, the buffer contains at most ~10 events per side. The watermark-driven expiration at line 166 is what makes this possible — without it, buffers would grow without bound.

The `TumblingWindowAggregator` (tested at line 141) extends this pattern by grouping join results into fixed time buckets. This is the "tumbling window" from DDIA Ch. 11 — non-overlapping, fixed-duration intervals where results accumulate until the window closes. The `advance_time` call triggers window closure, again driven by event-time watermarks rather than wall clocks.

## Topics to Explore

- [function] `stream-join-processor/stream_join_processor.py:_expire_events` — The core watermark-driven expiration logic that decides when buffered events are finalized or emitted as misses
- [file] `partitioned-log/partitioned_log.py` — The Kafka-style log this processor would consume from; understanding how `Message.timestamp` is assigned at produce time (line 143 uses `time.time()`) reveals the event-time/ingestion-time boundary
- [general] `watermark-computation-in-distributed-joins` — This implementation derives watermarks from a single stream; in a partitioned system, the watermark must be the minimum across all partitions, which is significantly harder
- [function] `stream-join-processor/stream_join_processor.py:advance_time` — How external watermark injection enables idle-stream handling and explicit trigger semantics
- [file] `unbundled-database/unbundled_database.py` — The CDC-driven derived systems use LSN-based ordering (a form of processing-time ordering) rather than event-time watermarks; comparing the two approaches clarifies when each model is appropriate

## Beliefs

- `watermark-monotonic-advance` — The watermark in `StreamJoinProcessor` is monotonically non-decreasing: it is set via `max(self._watermark, event.timestamp)` and `advance_time` rejects timestamps at or below the current watermark
- `late-event-drop-is-hard-cutoff` — Events with `timestamp < watermark - allowed_lateness` are unconditionally dropped and counted in `stats.late_events_dropped`; there is no secondary path to recover them
- `miss-emission-requires-expiration` — Outer join miss results (left-with-null-right or right-with-null-left) are only emitted during `_expire_events`, never at event arrival time; an unmatched event stays buffered until the watermark advances past the window boundary
- `join-determinism-from-event-time` — The join processor's output is fully determined by the sequence of `(stream_name, key, value, timestamp)` inputs and `advance_time` calls, with no dependency on wall-clock time, making it deterministically testable
- `buffer-bounded-by-window-plus-lateness` — Buffer size is bounded by the number of distinct keys with events within `window.duration` of the current watermark; `_expire_events` garbage-collects everything below that cutoff on every event or time advance

