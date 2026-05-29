# File: mapreduce-framework/mapreduce.py

**Date:** 2026-05-29
**Time:** 09:00

# `mapreduce-framework/mapreduce.py` — Single-Machine MapReduce Framework

## Purpose

This file implements a pedagogical single-machine MapReduce framework, modeling the programming model described in DDIA Chapter 10 (Batch Processing). It doesn't distribute work across machines — instead it simulates the MapReduce contract (map → shuffle/sort → reduce) using the local filesystem for intermediate data, exactly as a teaching tool should: it preserves the computational model while stripping away the distributed systems complexity.

The file owns two responsibilities: executing a single MapReduce job (`MapReduceJob`) and chaining multiple jobs into a pipeline (`MapReducePipeline`).

## Key Components

### `JobStats` (dataclass)
A plain data container tracking counters for a completed job: input/output record counts, worker counts, and wall-clock time. It's a read-only view — populated internally by `MapReduceJob.run()` and exposed via the `stats` property.

### `MapReduceJob`
The core abstraction. Accepts user-defined `mapper` and `reducer` functions with these contracts:

- **mapper**: `(key, value) → list[(key, value)]` — Takes one input record, emits zero or more intermediate key-value pairs.
- **reducer**: `(key, list[values]) → list[(key, value)]` — Takes a key and all values for that key, emits zero or more output records.
- **combiner** (optional): Same signature as reducer. Applied map-side to pre-aggregate before shuffle — the classic MapReduce optimization for associative/commutative operations (e.g., local summing before a global sum).

Constructor parameters `num_mappers` and `num_reducers` control the parallelism topology, though execution is sequential. `fault_tolerant` controls whether mapper/reducer exceptions are swallowed or propagated.

### `MapReducePipeline`
A lightweight chaining mechanism: stores an ordered list of `MapReduceJob` stages, feeds the output of each into the next. Uses builder-pattern (`add_stage` returns `self`).

## Patterns

**Three-phase execution model.** `run()` follows the canonical MapReduce pipeline:
1. **Map**: Split input → run mapper on each chunk → write partitioned intermediate files
2. **Shuffle/Sort**: Collect intermediate files by partition → sort by key → group by key
3. **Reduce**: Run reducer on each key-group → collect results

**Filesystem as intermediate storage.** Intermediate data is serialized as JSON to a temp directory (`map-{mapper_id}-part-{partition}.json`). This mirrors how real MapReduce uses a distributed filesystem for shuffle data. The naming convention encodes both the source mapper and target partition.

**Hash partitioning.** `hash(k) % num_reducers` assigns each intermediate key to a reducer partition — the same scheme used in Hadoop's `HashPartitioner`. This ensures all values for a given key land in the same reducer.

**Combiner as map-side optimization.** The combiner runs in `_run_mapper` after all pairs for a chunk are collected, applying group-by-key + reduce locally before writing to disk. This reduces shuffle volume.

## Dependencies

**Imports:** All stdlib — `json` for intermediate serialization, `os`/`shutil`/`tempfile` for filesystem management, `time` for elapsed timing, `groupby` from itertools for the key-grouping step, and typing for signatures.

**Imported by:** `test_mapreduce.py`, which exercises the framework with word count, numeric aggregation, and pipeline scenarios.

## Flow

`MapReduceJob.run()` orchestrates the full execution:

1. **Input normalization**: If `input_data` is a string, it's treated as a file path — lines are read and keyed by line number (1-indexed).
2. **Splitting**: `_split_input` distributes records across `num_mappers` chunks using integer division with remainder distribution (first `remainder` chunks get one extra record).
3. **Map phase**: For each chunk, `_run_mapper` applies the mapper to every record, partitions the output by `hash(key) % num_reducers`, optionally applies the combiner per partition, then writes each partition's pairs to a JSON file.
4. **Reduce phase**: For each partition, `_run_reducer` reads all intermediate files matching that partition suffix, sorts all pairs by key, groups with `itertools.groupby`, and calls the reducer per group.
5. **Final sort**: Results are sorted by key before returning.
6. **Cleanup**: The temp directory is always removed via `finally`.

For `MapReducePipeline.run()`: output of stage N becomes input of stage N+1. Each stage's `JobStats` are collected in `_stage_stats`.

## Invariants

- **All values for a key reach the same reducer.** Enforced by consistent hash partitioning in `_run_mapper`.
- **Reducer input is sorted by key.** `_run_reducer` sorts all pairs before grouping — `itertools.groupby` requires sorted input to group correctly.
- **Final output is sorted by key.** An explicit sort after the reduce phase.
- **Temp directory is always cleaned up.** The `try/finally` block in `run()` ensures `shutil.rmtree` executes even on exceptions.
- **Combiner is applied per-mapper, per-partition.** It operates on a single mapper's output for a single partition, not across mappers — this is correct, since combiners must be safe to apply to partial data.
- **Input splitting never drops records.** The remainder-distribution logic in `_split_input` guarantees `sum(len(chunk) for chunk in chunks) == len(data)`.

## Error Handling

Two modes controlled by `fault_tolerant`:

- **`fault_tolerant=False` (default):** Exceptions from mapper or reducer propagate immediately, aborting the job. The `finally` block still cleans up the temp directory.
- **`fault_tolerant=True`:** Mapper exceptions skip the failing record (`continue`). Reducer exceptions skip the failing key group. In both cases, the error is silently swallowed — no logging, no error collection. This means partial results are returned without any indication of data loss.

There's no retry logic — fault tolerance here means "skip and continue," not "retry until success." This is a simplification; real MapReduce frameworks would re-execute failed tasks on different workers.

## Topics to Explore

- [file] `mapreduce-framework/test_mapreduce.py` — See how mapper/reducer functions are written and what job configurations are tested (word count, combiners, pipelines, fault tolerance)
- [file] `batch-word-count/pipeline.py` — Compare this general-purpose framework with the specialized batch word-count implementation; likely a simpler, single-purpose batch job
- [file] `map-side-join/map_side_joins.py` — Map-side joins are a key MapReduce pattern from DDIA Chapter 10; see how join logic is implemented in the mapper without a reduce phase
- [general] `combiner-correctness` — Combiners must be associative and commutative to be correct (e.g., sum works, average doesn't). The framework doesn't enforce this — worth understanding why and what breaks if violated
- [general] `hash-partitioning-skew` — `hash(k) % num_reducers` can produce skew with non-uniform key distributions; contrast with the range-partitioning implementation in this repo

## Beliefs

- `mr-hash-partitions-keys` — MapReduceJob assigns intermediate keys to reducer partitions using `hash(key) % num_reducers`, so all values for a given key always reach the same reducer
- `mr-combiner-is-map-side-only` — The combiner runs inside `_run_mapper` on each mapper's output independently; it never sees data from other mappers
- `mr-intermediate-data-uses-json-files` — Intermediate shuffle data is serialized as JSON files in a temp directory with naming convention `map-{mapper_id}-part-{partition}.json`
- `mr-fault-tolerant-silently-drops` — When `fault_tolerant=True`, mapper and reducer exceptions are caught and silently skipped with no logging or error reporting, producing partial results without indication of data loss
- `mr-groupby-requires-presort` — `_run_reducer` sorts pairs by key before calling `itertools.groupby`, which is required because `groupby` only groups consecutive elements with the same key

