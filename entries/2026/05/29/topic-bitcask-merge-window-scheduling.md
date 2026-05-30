# Topic: The paper describes time-window-based merge scheduling to avoid compaction during peak load — neither implementation models this, but it's critical for production use

**Date:** 2026-05-29
**Time:** 12:32

# Time-Window-Based Merge Scheduling: A Production Gap

## What the Implementations Actually Do

Both LSM tree implementations in this repo trigger compaction **synchronously and unconditionally** — the exact opposite of time-window-aware scheduling.

### `log-structured-merge-tree/lsm.py`

The `LSMTree` constructor at **line 202-208** accepts a `compaction_threshold` (default 4). Compaction fires inline during writes:

```python
# line 315-317
if len(self._sstables) >= self._compaction_threshold:
    self.compact()
```

This means a `put()` or `delete()` call that happens to tip the SSTable count over the threshold will block the caller while `compact()` runs a full k-way merge (**line 319, line 323**). There's no notion of "now is a bad time" — the write that triggers threshold just pays the cost.

### `sstable-and-compaction/sstable.py`

The `CompactionManager` (visible in tests at **test_sstable.py:51-56**) uses the same pattern: `needs_compaction()` checks a count threshold (`min_threshold=2` for size-tiered, `l0_compaction_trigger=2` for leveled), and `run_compaction()` executes immediately when called. The strategy selection (size-tiered vs. leveled) controls *which* SSTables merge, but not *when* the merge happens relative to system load.

## What DDIA Describes and Why It Matters

Kleppmann's discussion of LSM compaction highlights a fundamental tension: compaction competes with foreground reads and writes for disk I/O bandwidth. In production systems like Cassandra and RocksDB, this manifests as:

1. **Write stalls** — if compaction can't keep up, the number of L0 SSTables grows, reads slow down (more files to check), and eventually writes must be throttled or blocked.
2. **Latency spikes** — a large compaction running during peak traffic causes p99 latency to spike because the disk is saturated with merge I/O.

Time-window-based scheduling addresses this by deferring non-urgent compaction to off-peak hours (e.g., overnight), rate-limiting compaction I/O during peak periods, or using separate I/O priorities. Cassandra's `compaction_throughput_mb_per_sec` and RocksDB's `rate_limiter` are real-world examples.

## What's Missing From Both Implementations

Neither implementation has any of the following, all of which would be needed for production-viable compaction scheduling:

- **A clock or load signal** — no concept of current time, request rate, or I/O utilization exists anywhere in either module. The grep for `time|window|peak|load|schedule` across the repo (**grep_time_window**) returns zero hits in either LSM module.
- **Asynchronous compaction** — both `compact()` methods run synchronously in the caller's thread. A production system would run compaction on a background thread with configurable priority.
- **Rate limiting** — no throttle on compaction I/O throughput. A merge of large SSTables will consume all available disk bandwidth.
- **Deferred scheduling** — no mechanism to say "compaction is needed but should wait." The check at `lsm.py:316` is a hard gate: threshold met → compact now.

## Why This Gap Is Critical

For a teaching implementation, immediate compaction is fine — it demonstrates the merge logic clearly. But any team porting this to production must understand that the *scheduling policy* around compaction is as important as the merge algorithm itself. Without it:

- Tail latencies become unpredictable under load
- Write throughput can drop to zero during large compactions
- The system has no way to prioritize user-facing work over background maintenance

This is a case where the algorithm is correct but the operational envelope is absent.

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:compact` — The synchronous k-way merge that blocks writers; understanding its I/O pattern shows why scheduling matters
- [function] `sstable-and-compaction/sstable.py:CompactionManager` — Implements strategy selection (size-tiered vs. leveled) but not scheduling; compare how Cassandra layers scheduling on top of strategy
- [general] `rocksdb-rate-limiter` — RocksDB's `rate_limiter` is the canonical example of throttling compaction I/O to protect foreground latency
- [general] `cassandra-compaction-windows` — Cassandra's `TimeWindowCompactionStrategy` and `compaction_throughput_mb_per_sec` show how time-awareness is layered into a real LSM system
- [file] `log-structured-merge-tree/test_lsm.py` — Tests verify compaction correctness but never test behavior under concurrent load, which is where scheduling becomes observable

## Beliefs

- `lsm-compaction-is-synchronous` — `LSMTree.compact()` in `lsm.py:319` runs synchronously in the caller's thread, blocking all reads and writes until the merge completes
- `compaction-trigger-is-count-only` — Both implementations trigger compaction solely on SSTable count thresholds (`compaction_threshold` at `lsm.py:208`, `min_threshold`/`l0_compaction_trigger` in `sstable.py`), with no load, time, or I/O awareness
- `no-compaction-rate-limiting` — Neither implementation throttles compaction I/O throughput, meaning a large merge will saturate disk bandwidth without regard to concurrent foreground operations
- `compaction-strategy-vs-scheduling-decoupled` — The `sstable.py` `CompactionManager` separates *which* SSTables to merge (strategy) from the caller's decision of *when* to merge, but neither the manager nor any caller implements scheduling logic

