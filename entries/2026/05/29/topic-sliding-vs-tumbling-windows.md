# Topic: This module implements tumbling windows but uses a sliding-style symmetric time check for join matching; explore how a true sliding window aggregator would differ

**Date:** 2026-05-29
**Time:** 14:20

# Tumbling Windows with Sliding-Style Join Matching

## The Core Tension

This module has two windowing strategies operating at different levels, and they follow different semantics — which is worth understanding because confusing the two is a common source of bugs in stream processing systems.

### The Join Match: Sliding-Style Symmetric Check

At `stream_join_processor.py:50-51`, `TimeWindow.contains()` uses:

```python
def contains(self, t1: float, t2: float) -> bool:
    return abs(t1 - t2) <= self.duration
```

This is **symmetric and event-centered**. For a window duration of 5 seconds, an event at `t=100` can match any event in the opposite stream with a timestamp in `[95, 105]`. The window "slides" with each event — it's anchored to the event's own timestamp, not to any fixed grid. This is exactly how interval joins work in systems like Kafka Streams or Flink: each event defines its own match range.

The join processor uses this at line 129 when scanning the opposite buffer:

```python
if self._window.contains(event.timestamp, other.event.timestamp):
```

Every buffered event in the opposite stream with the same key gets checked against this symmetric window. This means event A at `t=100` matching event B at `t=104` is identical to B matching A — the directionality doesn't matter.

### The Aggregator: Tumbling (Non-Overlapping) Windows

The `TumblingWindowAggregator` (line 225+) takes a completely different approach. From the test at `test_stream_join_processor.py:141-152`:

```python
agg = TumblingWindowAggregator(10.0, lambda key, results: len(results))
agg.add(JoinResult("k1", None, None, 3.0))   # -> window [0, 10)
agg.add(JoinResult("k1", None, None, 7.0))   # -> window [0, 10)
agg.add(JoinResult("k1", None, None, 13.0))  # -> window [10, 20)
```

Here, time is divided into a **fixed grid** of non-overlapping buckets: `[0,10)`, `[10,20)`, `[20,30)`, etc. Each join result is assigned to exactly one bucket based on its `join_timestamp`. Windows close when the watermark advances past their end boundary.

### Why This Combination Works (But Is Subtle)

The design composes two operations: first, find pairs using an interval-join semantic (sliding), then aggregate the results into tumbling buckets. This is valid — the join phase produces `JoinResult` objects with a `join_timestamp` of `max(left.timestamp, right.timestamp)` (line 139), and the aggregation phase bins those results into fixed windows.

But a developer should be aware that **the join window and aggregation window are independent parameters serving different purposes**. A 10-second join window means "events up to 10 seconds apart can match." A 10-second tumbling window means "aggregate results into 10-second buckets." Using the same duration for both is a convenience, not a requirement.

## How a True Sliding Window Aggregator Would Differ

A sliding (or "hopping") window aggregator differs from tumbling in three fundamental ways:

### 1. Overlapping Windows
A sliding window is defined by two parameters: **window size** and **slide interval** (or "hop"). With a 10-second window and 2-second slide, you'd get windows `[0,10)`, `[2,12)`, `[4,14)`, etc. A single event at `t=7` would belong to **five** windows simultaneously: `[0,10)`, `[2,12)`, `[4,14)`, `[6,16)` (the ones whose range contains 7). The tumbling aggregator here assigns each result to exactly one window — that's the defining constraint of tumbling (slide == size).

### 2. Memory and State Management
The tumbling aggregator can discard a window's state the moment it closes. A sliding aggregator must maintain state for all windows that overlap with the current time. With a 10-second window and 1-second slide, you'd have ~10 open windows at any moment, each holding its own accumulator. This multiplies memory by `window_size / slide_interval`.

An efficient sliding window aggregator typically uses a **complementary pairs** or **pane-based** approach (as described in DDIA's discussion of windowed operations): split the window into non-overlapping "panes" aligned to the slide interval, aggregate within each pane, then combine panes to produce each window's result. This avoids storing full event lists in every overlapping window.

### 3. Output Cadence
The tumbling aggregator emits one result per key per window period. A sliding aggregator emits a result at every slide boundary — so with a 2-second slide, you get 5x more output than a 10-second tumble. The current `advance_time` method (line 242) would need to emit results for every slide interval the watermark crosses, not just for fully-closed windows.

### What the Implementation Would Look Like

```python
class SlidingWindowAggregator:
    def __init__(self, window_size: float, slide_interval: float, agg_fn):
        self.window_size = window_size
        self.slide = slide_interval
        # Pane-based: accumulate into slide-aligned panes
        # key -> {pane_start: partial_aggregate}
        self._panes: dict[str, dict[float, Any]] = defaultdict(dict)

    def _pane_for(self, timestamp: float) -> float:
        return (timestamp // self.slide) * self.slide

    def add(self, result: JoinResult):
        pane = self._pane_for(result.join_timestamp)
        # accumulate into pane

    def advance_time(self, timestamp: float):
        # For each slide boundary crossed:
        #   combine panes in [boundary - window_size, boundary)
        #   emit the combined aggregate
        #   discard panes older than the oldest open window
```

The key architectural difference: the tumbling aggregator's `advance_time` walks through closed windows linearly, but a sliding aggregator must **re-combine panes** for each window it emits, since adjacent windows share most of their panes.

### The Expiration Check Connects Both Models

Notice that `_expire_events` at line 166 uses `cutoff = self._watermark - self._window.duration`, which is a **one-sided** check (is the event older than the window behind the watermark?). This is correct for buffer management — you only need to keep events that could still match future arrivals. But the *matching* check at line 51 is symmetric (`abs(t1 - t2)`). The asymmetry is intentional: expiration is about what's reachable from future events (always forward in time), while matching is about whether two existing events are close enough (either direction).

A sliding window aggregator wouldn't change the join semantics at all — it would only change how the *results* are grouped for downstream consumption.

## Topics to Explore

- [function] `stream-join-processor/stream_join_processor.py:_expire_events` — How watermark-driven expiration interacts with the symmetric join check; the cutoff is one-sided even though matching is symmetric
- [function] `stream-join-processor/stream_join_processor.py:TumblingWindowAggregator` — Full implementation of window assignment and closing logic, including how `advance_time` iterates closed windows
- [general] `pane-based-sliding-windows` — The optimization technique (from the Dataflow/Beam model) where sliding windows decompose into non-overlapping panes to avoid redundant computation
- [general] `session-windows-vs-fixed-windows` — Session windows (gap-based, data-driven boundaries) represent yet another windowing model that would interact differently with this join processor's event-centered matching
- [file] `stream-join-processor/test_stream_join_processor.py` — Test cases 12-13 cover tumbling aggregation and buffer cleanup; adding sliding window tests would reveal the output multiplier effect

## Beliefs

- `join-uses-symmetric-interval-not-tumbling` — `TimeWindow.contains()` uses `abs(t1 - t2) <= duration`, making join matching a symmetric interval check centered on each event, not aligned to any fixed time grid
- `tumbling-aggregator-assigns-each-result-to-one-window` — `TumblingWindowAggregator` assigns each `JoinResult` to exactly one non-overlapping time bucket via floor division on `join_timestamp`, so no result is counted twice
- `join-window-and-aggregation-window-are-independent` — The join `TimeWindow` duration controls which event pairs can match, while the `TumblingWindowAggregator` window size controls result grouping; these can differ without violating any invariant
- `expiration-cutoff-is-one-sided-despite-symmetric-matching` — `_expire_events` uses `watermark - duration` as a one-sided cutoff for buffer cleanup, while `contains()` is symmetric; this is correct because future events can only arrive at or after the watermark

