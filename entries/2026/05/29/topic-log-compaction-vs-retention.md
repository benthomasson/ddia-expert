# Topic: How compaction (keep latest per key) and retention (time/size truncation) interact — they can run independently but both mutate `_base_offsets`

**Date:** 2026-05-29
**Time:** 13:40

I'll work from the observations provided. They contain the first 200 lines of the implementation — enough to cover both `truncate` and `compact_partition` in full.

---

# How Compaction and Retention Interact via `_base_offsets`

## The Shared Mutable State

Both compaction and retention modify the same per-partition field: `_base_offsets[partition]`. This list, initialized to `[0] * num_partitions` at `partitioned_log.py:52`, tracks the logical offset of the first message still present in each partition. It's what consumers use to know where valid data begins (via `earliest_offset` at line 93).

The two operations are **logically independent** — one removes old messages by time/size, the other deduplicates by key — but they converge on this single bookkeeping field.

## Retention: `truncate()` (lines 98–104)

```python
def truncate(self, partition: int, count: int):
    log = self._partitions[partition]
    actual = min(count, len(log))
    self._partitions[partition] = log[actual:]
    self._base_offsets[partition] += actual
```

Truncation is straightforward: it removes `count` messages from the **front** of the partition. The `_base_offsets` update is a simple **addition** — the base moves forward by exactly as many messages as were dropped. This is safe because truncation only removes a contiguous prefix; offsets in the remaining messages are untouched and still monotonically increasing from the new base.

## Compaction: `compact_partition()` (lines 106–128)

```python
def compact_partition(self, partition: int) -> int:
    log = self._partitions[partition]
    # ...keep only latest occurrence per key...
    if removed > 0:
        if new_log:
            self._base_offsets[partition] = new_log[0].offset
        else:
            self._base_offsets[partition] = self._base_offsets[partition] + len(log)
        self._partitions[partition] = new_log
    return removed
```

Compaction is more complex. It scans all messages, keeps only the **latest** message for each key (plus all keyless messages), and rebuilds the log. The `_base_offsets` update is an **assignment** — it's set to the offset of the first surviving message. This is necessary because compaction removes messages from **arbitrary positions**, not just the front. The old base offset might belong to a message that was just discarded.

## The Interaction Problem

Because both operations mutate `_base_offsets`, the **order** in which they run matters:

1. **Truncate then compact**: Truncation advances `_base_offsets` by `+= actual`. Compaction then overwrites it with `new_log[0].offset`. If compaction removed the message that was previously at the front (because it was a stale duplicate), the base offset is correctly recalculated. This ordering is safe.

2. **Compact then truncate**: Compaction sets `_base_offsets` to the first surviving message's offset. Truncation then advances it by `+= actual`. This is also safe because truncation operates on the current log contents, and `actual = min(count, len(log))` bounds it correctly.

3. **Concurrent execution**: There's no locking. If both ran simultaneously on the same partition, they'd race on both `_partitions[partition]` (the list itself) and `_base_offsets[partition]`. One could overwrite the other's update. This implementation assumes **single-threaded** execution.

## Key Asymmetry

The critical insight is that the two operations update `_base_offsets` **differently**:

| Operation | Update style | Reason |
|-----------|-------------|--------|
| `truncate` | `+= actual` (relative) | Only removes a contiguous prefix; the shift is predictable |
| `compact_partition` | `= new_log[0].offset` (absolute) | Removes from arbitrary positions; must recalculate from surviving data |

This means truncation **composes** with whatever the current base is (it's additive), while compaction **replaces** it (it's absolute). If you need to reason about what `earliest_offset` returns after both operations, compaction's assignment is the dominant write — it doesn't care what the base was before, only what survived.

## Offset Gaps After Compaction

After compaction, the offsets in the log are **no longer contiguous**. The `read()` method (lines 80–92) handles this correctly by scanning for the first message with `msg.offset >= offset` rather than computing an index arithmetically. This linear scan is necessary precisely because compaction creates offset gaps.

After truncation, offsets remain contiguous within the surviving log — truncation only removes a prefix. But if compaction ran first, the truncated log may already have gaps.

## What the Broker Does With These

The observations are cut off at line 200, so the Broker-level retention policy (the code that decides *when* to call `truncate` based on `retention_ms` and `max_log_size` configured at lines 44–45) isn't visible. The Topic stores these config values but the enforcement logic — the code that checks wall-clock time or byte counts and calls `truncate` accordingly — likely lives in the Broker or a background maintenance method further in the file.

---

## Topics to Explore

- [function] `partitioned_log.py:Topic.read` — How the linear scan handles offset gaps left by compaction, and whether this becomes a performance concern at scale
- [file] `partitioned_log.py` (lines 200–481) — The Broker class likely contains the retention enforcement loop that calls `truncate` based on `retention_ms` and `max_log_size`
- [function] `partitioned_log.py:Topic.append` — How offset assignment (`log[-1].offset + 1 if log else self._base_offsets[partition]`) stays correct after compaction creates gaps
- [general] `concurrent-compaction-safety` — Whether any external caller (Broker, background thread) could trigger both `truncate` and `compact_partition` on the same partition simultaneously, and what breaks if so
- [function] `test_partitioned_log.py:test_log_compaction` — Likely exists in the untested portion (lines 200+) and would show expected behavior when both operations interact

## Beliefs

- `truncate-advances-base-offset-additively` — `Topic.truncate` updates `_base_offsets[partition]` with `+= actual`, a relative shift, because it only removes a contiguous prefix
- `compact-sets-base-offset-absolutely` — `Topic.compact_partition` overwrites `_base_offsets[partition]` with the first surviving message's offset, discarding whatever the previous base was
- `compaction-preserves-original-offsets` — After `compact_partition`, surviving messages retain their original offset values, creating non-contiguous gaps in the offset sequence
- `read-uses-linear-scan-for-offset-gaps` — `Topic.read` finds the start position by scanning for `msg.offset >= offset` rather than computing an array index, which is necessary because compaction creates offset holes
- `no-concurrency-protection-on-base-offsets` — Neither `truncate` nor `compact_partition` holds a lock; concurrent calls on the same partition would race on both `_partitions` and `_base_offsets`

