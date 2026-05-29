# File: batch-word-count/pipeline.py

**Date:** 2026-05-29
**Time:** 08:53

## Purpose

`pipeline.py` implements a composable batch processing framework modeled after Unix pipes. Each processing step is a `Stage` that consumes an iterator and yields records, and the `Pipeline` class chains them together into a lazy, pull-based dataflow. The canonical use case is word counting (ReadLines → Tokenize → Count → Sort), but the framework is general-purpose — it corresponds to the batch processing concepts from DDIA Chapter 10 (MapReduce and beyond), distilled into a single-process, iterator-based design.

## Key Components

### `Stage` (base class)

The abstract unit of work. Every stage implements `process(input_iter)` as a generator that consumes input records and yields output records. The base class provides two pieces of infrastructure:

- **`_tracked_process`** — wraps `process` to collect timing and record counts. It subtracts downstream time (time spent in the *next* stage's processing after a `yield`) from this stage's elapsed time, giving per-stage wall-clock attribution.
- **`_count_input`** — wraps the input iterator to increment `records_in`. Stages must explicitly call this (e.g., `self._count_input(input_iter)`) to opt into input counting — not all do (`ReadLines` and `Count` skip it in different ways).

### Concrete Stages

| Stage | Input | Output | Materialization |
|-------|-------|--------|-----------------|
| `ReadLines` | file path or list of strings | `(line_number, text)` | Streaming |
| `Tokenize` | `(key, text)` | `(word, 1)` | Streaming |
| `Count` | `(key, value)` | `(key, total)` | **Full** — accumulates all keys in a `defaultdict` before yielding |
| `Sort` | any tuple | same tuples, sorted | **Full** (in-memory) or **external** (spills to disk) |
| `Filter` | any record | same record (if predicate passes) | Streaming |
| `TopN` | any tuple | top N tuples by key or value | Partial — maintains a min-heap of size N |
| `Partition` | any record | same record (pass-through), plus side-effect writes | Streaming with file I/O side effects |
| `FlatMap` | any record | zero or more records from `fn(record)` | Streaming |

### `Pipeline`

The orchestrator. `add_stage()` appends stages and returns `self` for chaining. Three execution modes:

- **`run()`** — materializes the full result as a list.
- **`run_to_file()`** — streams results to a JSONL file without holding them all in memory.
- **`run_lazy()`** — returns a generator; stats are finalized in the `finally` block when the generator is exhausted or garbage-collected.

## Patterns

**Pull-based iterator chaining.** Each stage wraps the previous stage's output iterator — the pipeline is a nested generator chain evaluated lazily from the tail. This is the Unix pipe model: data flows record-by-record, and only materializing stages (`Count`, `Sort`) buffer data.

**External merge sort.** `Sort` implements a textbook external sort: when the in-memory buffer hits `memory_limit`, it flushes a sorted chunk to a temp file as JSONL. After all input is consumed, it uses `heapq.merge` over the chunks. The `KeyedRecord` wrapper class with custom `__lt__` handles both ascending/descending order and stable sequencing via a `seq` tiebreaker.

**Downstream time subtraction.** `_tracked_process` measures elapsed time but subtracts time spent after each `yield` (which is time the downstream stage is processing). This gives accurate per-stage attribution in a lazy pipeline where stages interleave execution.

**Builder pattern.** `Pipeline.add_stage()` returns `self`, enabling `p.add_stage(A()).add_stage(B())` fluent construction.

## Dependencies

**Imports:** All stdlib — `heapq` for merge sort and TopN, `json` for serialization in Sort/Partition, `tempfile`/`os` for external sort spill files, `re` for punctuation stripping in Tokenize, `time` for stats, `defaultdict` for Count.

**Imported by:** `test_pipeline.py` and `tester_test_pipeline.py` — the test suite and its meta-test harness.

## Flow

A typical word-count pipeline executes as:

1. `Pipeline.run()` calls `_build_iterator()`, which nests the stages: `Sort._tracked_process(Count._tracked_process(Tokenize._tracked_process(ReadLines._tracked_process(...))))`.
2. `list(...)` pulls from the outermost iterator.
3. `Sort` pulls from `Count`, which pulls from `Tokenize`, which pulls from `ReadLines` — all lazily.
4. `Count` is a barrier: it consumes its entire input before yielding anything, so everything upstream of Count runs to completion before Sort sees any records.
5. After the iterator is exhausted (or errors), `_build_stats` collects per-stage metrics.

The `_build_iterator` method has special-case handling for the first stage: if it's a `ReadLines`, it uses its own source; otherwise, if `input_data` is provided, a temporary `ReadLines` is prepended (but *not* added to `self._stages`, so its stats aren't tracked).

## Invariants

- **Record format convention.** Stages communicate via tuples — most expect `(key, value)` pairs. There's no schema enforcement; a `Tokenize` after a `Sort` would break silently if the Sort output doesn't match `(key, text)` shape.
- **Sort stability.** External sort preserves insertion order for equal keys via the `seq` tiebreaker in `KeyedRecord.__lt__`.
- **Temp file cleanup.** `Sort.process` uses a `try/finally` to delete chunk files and the temp directory, even on error.
- **Partition file cleanup.** `Partition.process` closes all file handles in `finally`, but doesn't delete the files (they're the intended output).
- **Stats require complete execution.** `_build_stats` runs in a `finally`, but per-stage timing is only accurate if the iterator was fully consumed. A partially consumed `run_lazy()` generator will have incomplete `records_out` counts.

## Error Handling

Minimal and deliberate. There are no try/except blocks that catch and swallow errors — exceptions propagate up the generator chain naturally. The `finally` blocks in `Sort`, `Partition`, and the `Pipeline.run*` methods ensure cleanup (temp files, open file handles, stats collection) happens even on failure. `ReadLines` will raise `FileNotFoundError` if given a nonexistent path. The pipeline does not validate stage compatibility at construction time.

## Topics to Explore

- [file] `batch-word-count/test_pipeline.py` — Shows the expected usage patterns, edge cases, and the record format contracts between stages
- [function] `batch-word-count/pipeline.py:Sort.process` — The external merge sort is the most complex algorithm here; worth tracing through the chunk-spill and `heapq.merge` path with a concrete example
- [file] `mapreduce-framework/mapreduce.py` — Compare this single-process pipeline approach with the distributed MapReduce implementation in the same repo
- [general] `ddia-chapter-10-batch-processing` — The Unix philosophy of composable tools, sort-merge joins, and the distinction between streaming and materializing operators that this code demonstrates
- [function] `batch-word-count/pipeline.py:Stage._tracked_process` — The downstream-time-subtraction technique for accurate per-stage timing in a lazy pipeline is subtle and worth understanding in detail

## Beliefs

- `pipeline-lazy-pull-model` — Pipeline execution is pull-based: records flow only when the terminal consumer requests them, and stages interleave execution via nested generators
- `sort-external-merge` — Sort spills to disk when buffer exceeds `memory_limit` and uses `heapq.merge` with a `KeyedRecord` wrapper for stable k-way merge of sorted chunks
- `count-is-barrier` — Count materializes its entire input into a `defaultdict` before yielding any output, making it a pipeline barrier that forces all upstream stages to complete first
- `stage-timing-subtracts-downstream` — `_tracked_process` subtracts time spent after each `yield` (downstream processing) to attribute wall-clock time accurately to each stage
- `no-schema-validation` — The pipeline performs no compile-time or runtime validation that adjacent stages produce/consume compatible record formats; mismatched shapes fail at runtime with tuple-indexing errors

