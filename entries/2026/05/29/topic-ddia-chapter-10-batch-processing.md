# Topic: The Unix philosophy of composable tools, sort-merge joins, and the distinction between streaming and materializing operators that this code demonstrates

**Date:** 2026-05-29
**Time:** 12:53

# The Unix Philosophy in Batch and Stream Processing

## Composable Tools: The Pipeline as a Chain of Generators

The `batch-word-count/pipeline.py` file is the clearest expression of the Unix philosophy in this codebase. Each `Stage` is a standalone transformation — it consumes an iterator and yields records, exactly like a Unix command consuming stdin and writing to stdout. The `Pipeline` class composes stages by nesting their generators:

```
Sort._tracked_process(Count._tracked_process(Tokenize._tracked_process(ReadLines._tracked_process(...))))
```

This is `cat | tokenize | sort | uniq -c` rendered as Python generators. Data flows **record-by-record** through the chain, pulled lazily from the tail — the `Pipeline` never builds a central data structure. The three execution modes (`run()`, `run_lazy()`, `run_to_file()`) in `pipeline.py` mirror the Unix choice between materializing output, piping it, or redirecting to a file.

The `Pipeline.add_stage()` method returns `self` for fluent builder-style chaining. But critically, there is **no schema validation** between stages — just as Unix pipes pass untyped byte streams, the pipeline passes untyped tuples. A `Tokenize` stage after a `Sort` would break at runtime if the tuple shape doesn't match, but nothing prevents you from building that pipeline.

The `mapreduce-framework/mapreduce.py` takes this further with `MapReducePipeline`, which chains multiple `MapReduceJob` stages where the output of stage N becomes the input of stage N+1 — the same composition pattern at a coarser granularity.

## Sort-Merge Joins: Three Strategies, Same Result

`map-side-join/map_side_joins.py` implements the three map-side join strategies from DDIA Chapter 10, each trading different assumptions for performance:

| Strategy | Assumption | Mechanism |
|----------|-----------|-----------|
| `BroadcastHashJoin` | One dataset fits in memory | Build hash table from small side, probe with large side |
| `PartitionedHashJoin` | Both sides co-partitioned on join key | Hash-partition both, local hash join per partition |
| `SortMergeJoin` | Both sides sorted by join key | Linear two-pointer merge |

The `SortMergeJoin` is the textbook sort-merge: it checks sort order with `_is_sorted()`, sorts if needed, then walks two pointers forward. When duplicate keys appear, it collects all matching records from both sides and emits their **cartesian product** — every left record paired with every right record sharing that key.

The `compare_join_strategies()` function is the pedagogical payoff: it runs all three strategies on the same data and verifies they produce identical results (after normalizing away `_mapper_id` tags). This demonstrates DDIA's point that these are different implementations of the same logical operation, each optimal under different data assumptions.

All three share the same field-conflict resolution via `_merge_records()`: the join key appears exactly once in the output, and any other shared field names get `left_`/`right_` prefixes.

## Streaming vs. Materializing Operators

This is the deepest concept in the codebase, and it cuts across multiple modules. The key question is: **does an operator need to see all its input before producing any output?**

### The Spectrum in `pipeline.py`

The batch pipeline's stages fall into three categories:

| Category | Stages | Memory Behavior |
|----------|--------|-----------------|
| **Streaming** | `ReadLines`, `Tokenize`, `Filter`, `FlatMap`, `Partition` | O(1) memory — yield immediately per input record |
| **Partial materialization** | `TopN` | O(N) memory — bounded heap, yields only at end |
| **Full materialization** | `Count`, `Sort` | O(input) memory — must see everything before yielding anything |

`Count` is explicitly a **pipeline barrier**: it accumulates all keys in a `defaultdict` before yielding any output, which forces every upstream stage to run to completion. `Sort` is the same — you cannot emit sorted output until you've seen the last record. These are the operators that break the Unix illusion of smooth record-by-record flow.

`Sort` in `pipeline.py` mitigates this with **external merge sort**: when the in-memory buffer exceeds `memory_limit`, it spills sorted chunks to disk as JSONL, then uses `heapq.merge` for k-way merge across chunks. The `KeyedRecord` wrapper provides stable ordering via a `seq` tiebreaker. This is the same technique MapReduce uses for its shuffle phase — and it's why the test in `test_pipeline.py` verifies that external sort produces identical results to in-memory sort.

### Batch vs. Stream: Two Worlds of Materialization

The `mapreduce-framework/mapreduce.py` represents **full materialization** between phases. Intermediate data is serialized as JSON files (`map-{mapper_id}-part-{partition}.json`) and written to disk. The shuffle phase reads all intermediate files for a partition, sorts by key, groups with `itertools.groupby`, then feeds to the reducer. Every phase boundary is a materialization barrier — this is the fundamental limitation of MapReduce that dataflow engines like Spark address.

In contrast, `stream-join-processor/stream_join_processor.py` represents **continuous, incremental processing**. The `StreamJoinProcessor` never waits for "all input" because in a stream there is no "all." Instead it uses:

- **Event-time watermarks** — the highest timestamp seen — to decide when events are "expired"
- **Probe-on-arrival** — when an event arrives, it's buffered and immediately probed against the opposite buffer (a nested-loop join scoped to matching keys)
- **Deferred miss emission** — outer-join misses are emitted only at expiration (`watermark - window_duration`), not eagerly, because a match might still arrive

The stream processor's buffers (`defaultdict(list)` keyed by join key) are a form of materialization, but **bounded** — events expire and are removed. This is the fundamental tradeoff: batch operators materialize everything for correctness; stream operators bound materialization for latency, accepting that late events may be dropped.

### The Unbundled Database: Composition Through Logs

The `unbundled-database/unbundled_database.py` synthesizes both paradigms. The `WriteAheadLog` is a streaming, append-only data source. The `StorageEngine` is a materializing consumer — it maintains the full current state. The `CDCStream` fans changes out to derived systems (`SecondaryIndex`, `MaterializedView`, `FullTextSearch`), each of which tracks its own position independently (analogous to Kafka consumer group offsets).

This is the Unix philosophy applied at the system level: each component has one job, communicates through a shared log (the "pipe"), and can be replaced independently. The `DerivedSystem` interface is the composable stage contract — implement `process_event()`, `rebuild()`, and `get_state()`, and you plug into the pipeline.

The `snapshot_and_stream()` method on `CDCStream` solves the "new consumer catch-up" problem by synthesizing insert events from current storage state, then subscribing for future events — a neat bridge between the materialized world (full state) and the streaming world (incremental changes).

---

## Topics to Explore

- [function] `batch-word-count/pipeline.py:Sort.process` — The external merge sort implementation with chunk spilling and `heapq.merge`; the most algorithmically complex stage and the clearest illustration of why sort is a materializing operator
- [function] `stream-join-processor/stream_join_processor.py:_expire_events` — The watermark-driven expiration and miss emission logic; trace how it interacts with one-to-many matches and the `matched` flag to understand bounded materialization in streams
- [function] `unbundled-database/unbundled_database.py:CDCStream.snapshot_and_stream` — The catch-up mechanism for late-joining consumers; compare with Kafka's compacted topic approach for bridging batch and stream
- [general] `combiner-correctness` — The MapReduce combiner in `mapreduce.py:_run_mapper` is a streaming optimization (pre-aggregate before shuffle) that only works for associative/commutative operations; explore what breaks when this contract is violated
- [file] `map-side-join/test_map_side_joins.py` — See the edge cases exercised across all three join strategies, particularly how duplicate keys, missing keys, and field conflicts are handled

## Beliefs

- `count-sort-are-materialization-barriers` — `Count` and `Sort` in `pipeline.py` must consume their entire input before yielding any output, making them pipeline barriers that break the otherwise record-by-record streaming flow
- `sort-merge-join-cartesian-on-duplicates` — `SortMergeJoin` collects all records with the same key from both sides and emits every combination (cartesian product), which means join output can be quadratic in the number of duplicate keys
- `mapreduce-materializes-at-phase-boundaries` — MapReduce in `mapreduce.py` writes intermediate data to JSON files between map and reduce phases, creating a full materialization barrier that prevents streaming from mapper to reducer
- `stream-join-bounds-materialization-via-watermarks` — `StreamJoinProcessor` expires buffered events when they fall below `watermark - window_duration`, bounding memory at the cost of potentially dropping late-arriving matches
- `unbundled-db-composes-via-log` — The unbundled database wires independent subsystems (WAL, storage, CDC, derived systems) through a shared log with independent consumer positions, applying Unix-style composition at the system architecture level

