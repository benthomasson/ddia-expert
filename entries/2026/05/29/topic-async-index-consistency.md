# Topic: Explore what happens when you query a term-partitioned DB with `async_index=True` before calling `flush_index` — stale reads are the real-world consequence DDIA warns about

**Date:** 2026-05-29
**Time:** 14:11

The file is in the target repo. Let me work from the observations provided — I have enough to trace the full flow.

---

# Stale Reads in Term-Partitioned Async Indexes

## The Setup

In DDIA Chapter 6, Kleppmann explains that term-partitioned (global) secondary indexes offer efficient reads — you only query *one* partition for a given index term — but writes become more expensive because a single document write may need to update index entries on *multiple* partitions. The real-world solution? **Make the index update asynchronous.** The implementation in `secondary-index-partitioning/secondary_index_partitioning.py` models this tradeoff directly.

## What Happens Step by Step

### 1. You create the DB with `async_index=True`

```python
db = TermPartitionedDB(4, ["color"], async_index=True)
```

At line 138–143, the constructor stores `self.async_index = async_index` and initializes `self._pending: list[tuple] = []`. That pending list is the write-ahead queue — index operations that have been *recorded* but not yet *applied* to the global index.

### 2. You write a document

```python
db.put("p1", {"color": "red"})
```

During `put()`, the document itself is stored immediately in the correct partition (line ~200, via `_partition_for(pk)`). But when it's time to update the global index, the code hits this branch at line 203:

```python
if self.async_index:
```

Instead of calling `_apply_index_op("add", field, value, pk)` right away, the operation is appended to `self._pending`. The index entry `"red" → {"p1"}` is **not written** to any partition's `global_index` yet.

### 3. You query — and get nothing

```python
result = db.query_by_field("color", "red")
assert len(result.data) == 0  # stale!
```

The test at line 187–190 (`test_stale_before_flush`) proves this directly. `query_by_field` looks up `"red"` in the global index of the partition responsible for that term. The index is empty — the operation is still sitting in `_pending`. **The document exists but is invisible to secondary index queries.**

This is the core phenomenon: your data is there (a `get("p1")` by primary key works fine), but any query that goes through the secondary index returns stale results.

### 4. You flush, and the world catches up

```python
count = db.flush_index()
result = db.query_by_field("color", "red")
assert len(result.data) == 1  # now visible
```

`flush_index()` at line 294 drains `self._pending`, calling `_apply_index_op()` for each queued operation. After flush, the global index on the appropriate partition contains `"red" → {"p1"}`, and queries work.

### 5. Updates are also deferred

The test at line 201 (`test_async_update`) shows the compound case: you write `color=red`, then update to `color=blue`, then flush. Both the remove-old and add-new operations queue in `_pending` (lines 214 and 203 respectively). Only after `flush_index()` does the index correctly reflect the current state.

## Why This Matters (The DDIA Connection)

This models what real systems like Amazon DynamoDB's Global Secondary Indexes do. Kleppmann (Chapter 6, "Partitioning and Secondary Indexes") warns:

> Updates to global secondary indexes are often asynchronous... a read from the index may not have caught up with the latest writes.

The implementation makes this concrete with a measurable gap: between `put()` and `flush_index()`, there is a window where primary-key reads are consistent but secondary-index queries return stale results. In production, this window is bounded by replication lag. Here it's bounded by explicit `flush_index()` calls, making the staleness deterministic and testable.

The `_pending` queue is the simulation of what would be a cross-partition replication log in a distributed system. The `flush_index()` call simulates the background process that applies those updates. The test at line 333–337 in the compare test class even sets up an async DB alongside a sync one to compare their behavior.

## The Tradeoff Made Explicit

| | Sync index (`async_index=False`) | Async index (`async_index=True`) |
|---|---|---|
| **Write latency** | Higher — must update index partitions in same call | Lower — just append to `_pending` |
| **Read consistency** | Always current | Stale until `flush_index()` |
| **`_apply_index_op`** | Called in `put()`/`delete()` | Called only in `flush_index()` |

This is exactly the consistency-vs-performance tradeoff DDIA describes. You trade read freshness for write speed.

## Topics to Explore

- [function] `secondary-index-partitioning/secondary_index_partitioning.py:_apply_index_op` — The mechanism that actually mutates the global index; understanding how it routes by `_term_partition(value)` shows why a single document write fans out to multiple index partitions
- [function] `secondary-index-partitioning/secondary_index_partitioning.py:query_range` — Range queries on range-partitioned global indexes (line 269) show the read-side advantage that justifies the write-side complexity of term partitioning
- [function] `secondary-index-partitioning/secondary_index_partitioning.py:compare_strategies` — Runs both document-partitioned and term-partitioned DBs side by side with identical workloads, making the partition-touch-count tradeoff empirically visible
- [general] `scatter-gather-vs-single-partition-query` — Document-partitioned queries always touch all partitions (scatter/gather, line 98: `partitions_touched == 4`), while term-partitioned queries touch at most one; this is the fundamental read efficiency motivation for global indexes
- [file] `leaderless-replication/dynamo.py` — The read-repair mechanism at line 153 is another strategy for handling staleness in distributed indexes, complementing the async-flush approach

## Beliefs

- `async-index-defers-to-pending-queue` — When `async_index=True`, `TermPartitionedDB.put()` and `delete()` append index operations to `self._pending` instead of calling `_apply_index_op()`, making the document immediately stored but invisible to secondary index queries
- `flush-index-drains-pending` — `flush_index()` iterates `self._pending` and calls `_apply_index_op()` for each entry, returning the count of operations applied; it is the only mechanism that materializes async index updates
- `primary-key-reads-unaffected-by-async` — `get(pk)` reads directly from `partitions[pid].documents` and is never affected by `async_index` mode — staleness only applies to `query_by_field` which reads from `global_index`
- `term-partition-query-touches-one-partition` — `query_by_field` in `TermPartitionedDB` computes `_term_partition(value)` and reads from a single partition's `global_index`, touching at most 1 partition vs the document-partitioned approach which always touches all `num_partitions`

