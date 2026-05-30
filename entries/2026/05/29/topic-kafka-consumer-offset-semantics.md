# Topic: DDIA Ch. 11 discussion of exactly-once vs at-least-once delivery — this implementation's `auto_commit` in `poll()` creates at-least-once semantics with potential duplicate processing on crash

**Date:** 2026-05-29
**Time:** 13:43

The observations include the full `Consumer.__init__` and the key grep hits but don't include the body of `poll()` (lines 237–260). Let me work with what we have and note the gap.

---

## The At-Least-Once Gap in `auto_commit`

### The Core DDIA Concept

Chapter 11 of *Designing Data-Intensive Applications* frames message delivery guarantees around a fundamental question: **where does the offset commit happen relative to message processing?** There are three delivery semantics:

| Semantic | Offset committed... | Crash consequence |
|---|---|---|
| **At-most-once** | Before processing | Messages lost (skipped) |
| **At-least-once** | After processing | Messages reprocessed (duplicated) |
| **Exactly-once** | Atomically with processing | Neither lost nor duplicated |

The gap between "message received" and "offset committed" is where delivery guarantees live or die.

### How This Implementation Works

In `partitioned-log/partitioned_log.py`, the `Consumer` constructor (line 175) accepts an `auto_commit` flag:

```python
# line 182
auto_commit: bool = False,
```

This flag is stored at line 187 (`self._auto_commit = auto_commit`) and checked inside `poll()` at line 254:

```python
if self._auto_commit and result:
```

The `poll()` method (line 237) fetches messages from assigned partitions and — when `auto_commit` is enabled — commits offsets as part of the same call. The critical design question is the **ordering** within `poll()`:

1. Fetch messages from partitions, advancing internal `_offsets` (the `dict[tuple[str, int], int]` at line 190)
2. If `auto_commit` and results exist, call `self.commit()` — persisting `_offsets` to the broker
3. Return the messages to the caller

### Where Duplicates Come From

The caller's processing loop looks like this:

```python
while True:
    messages = consumer.poll()    # fetch + auto-commit
    for msg in messages:
        process(msg)              # application logic — writes to DB, updates state, etc.
```

**The at-least-once window** exists because offset commitment and message processing are two separate, non-atomic operations. Consider this crash timeline:

1. `poll()` returns messages at offsets 0, 1, 2
2. Application processes message 0 (e.g., writes a row to a database)
3. **CRASH** — before the next `poll()` can commit, or before the current `poll()`'s commit completed
4. On restart, a new consumer in the same group loads the last committed offset (say, 0)
5. Messages 0, 1, 2 are **all redelivered** — message 0 is processed a second time

The test at `test_partitioned_log.py:152` (`test_offset_commit_restore`) demonstrates this offset-resume mechanism: after commit and restart, a new consumer picks up exactly where the committed offset left off. That's the machinery that makes redelivery possible.

### Why Exactly-Once Is Hard

To achieve exactly-once, you'd need to **atomically** combine the offset commit with the processing side effect. DDIA describes two approaches:

1. **Idempotent processing** — make `process(msg)` safe to call twice (e.g., using a deduplication key derived from the message offset). Several other modules in this repo implement idempotency: `consistent-hashing/test_consistent_hashing.py:112` tests `test_duplicate_add_idempotent`, and `multi-leader-replication/test_multi_leader.py:64` tests `test_idempotency`.

2. **Transactional outbox / two-phase commit** — write the offset and the processing result in a single transaction. Kafka achieves this with its transactional producer API, which this implementation does not model.

### What's Missing From the Observations

I couldn't read the full `poll()` body (lines 237–260) due to the observation window ending at line 200. The exact commit timing within `poll()` — whether it commits the *current* batch's offsets or the *previous* batch's — determines whether the default failure mode is at-most-once (commit before processing) or at-least-once (commit tied to the next poll). The user's framing as "at-least-once" suggests the commit occurs within `poll()` but the processing happens after `poll()` returns, creating the classic window for redelivery on crash.

---

## Topics to Explore

- [function] `partitioned-log/partitioned_log.py:poll` — Read the full poll() body (lines 237–260) to confirm whether auto_commit commits current or previous batch offsets, and whether commit happens before or after the return
- [function] `partitioned-log/partitioned_log.py:commit` — Understand what state gets persisted to the broker and how a restarting consumer loads it
- [file] `stream-join-processor/stream_join_processor.py` — The stream join processor processes events with watermarks but has no offset commit mechanism at all — compare its failure semantics
- [general] `idempotent-consumer-pattern` — Several modules (consistent-hashing, multi-leader-replication, PBFT) implement idempotency checks; study how they deduplicate to understand the exactly-once alternative
- [function] `partitioned-log/partitioned_log.py:seek` — The seek API (tested at line 170) lets consumers manually rewind, which is the escape hatch when at-least-once reprocessing produces incorrect results

## Beliefs

- `auto-commit-not-default` — `Consumer` defaults to `auto_commit=False` (line 182), requiring explicit `commit()` calls; this makes at-least-once the opt-in behavior rather than the default
- `offset-tracking-per-partition` — Consumer tracks offsets as a `dict[tuple[str, int], int]` mapping (topic, partition) to offset (line 190), meaning commit granularity is per-partition, not per-message
- `no-transactional-commit` — The implementation provides no mechanism to atomically commit offsets together with processing side effects, ruling out exactly-once delivery without external idempotency
- `consumer-group-offset-resume` — A new consumer joining a group loads committed offsets from the broker (tested at line 152), which is the mechanism that enables both offset-based resume and duplicate redelivery on crash

