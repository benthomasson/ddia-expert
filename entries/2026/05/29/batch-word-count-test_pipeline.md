# File: batch-word-count/test_pipeline.py

**Date:** 2026-05-29
**Time:** 12:49

I'll work with the test file content provided to give you the explanation.

---

# `batch-word-count/test_pipeline.py`

## Purpose

This is a smoke-test script for a batch data processing pipeline — a simplified model of the kind of staged, composable processing you'd see in MapReduce or dataflow systems like Apache Beam. It validates that each pipeline stage (`ReadLines`, `Tokenize`, `Count`, `Sort`, `TopN`, `Filter`, `Partition`, `FlatMap`) composes correctly through the `Pipeline` orchestrator, and that auxiliary features (lazy evaluation, file I/O, stats tracking, external sort) work end-to-end.

It's a script, not a pytest suite — it runs top-to-bottom with `assert` statements and prints `ALL TESTS PASSED` at the end. This makes it a fast, zero-dependency validation tool.

## Key Components

There are no classes or functions defined here. The file is a linear test script that exercises the following imported components from `pipeline.py`:

| Component | Role in tests |
|-----------|--------------|
| `Pipeline` | Orchestrator — stages are added via `.add_stage()`, executed via `.run()`, `.run_lazy()`, or `.run_to_file()` |
| `ReadLines` | Source stage — accepts either a list of strings or a file path |
| `Tokenize` | Splits lines into words; supports `lowercase` and `strip_punctuation` options |
| `Count` | Aggregation stage — produces `(word, count)` pairs |
| `Sort` | Sorts by `"key"` or `"value"`, with `descending` flag; supports external sort via `memory_limit` |
| `TopN` | Returns the top N records by a given field |
| `Filter` | Predicate-based filtering over records |
| `Partition` | Writes records to per-partition files based on a `partition_fn` |
| `FlatMap` | One-to-many transformation — each input record can produce multiple output records |

## Patterns

**Composable pipeline pattern.** Each test constructs a `Pipeline`, adds stages linearly, and calls `.run()`. This mirrors the batch dataflow model from DDIA Chapter 10 — data flows through a DAG of transformations, each stage consuming the output of the previous one. The API is builder-style: `pipeline.add_stage(X)` chains stages in order.

**Script-as-test.** No test framework, no fixtures, no parametrization. Each test case is a block of imperative code with inline assertions. This is a deliberate choice for a reference implementation — it keeps the test readable as a usage example.

**Temp-file discipline.** Tests that touch the filesystem (`Partition`, file input, `run_to_file`) use `tempfile.TemporaryDirectory` or `tempfile.NamedTemporaryFile` with explicit cleanup in `finally` blocks, preventing leftover artifacts.

**Wildcard import.** `from pipeline import *` — appropriate here since the test file is exercising the public API surface of the module and needs every exported name.

## Dependencies

**Imports:**
- `pipeline` (local) — the entire public API via wildcard import
- `tempfile`, `os` — stdlib, for filesystem-based test cases

**Imported by:** Nothing. This is a leaf test script. The sibling `tester_test_pipeline.py` likely wraps or validates this file but does not import from it.

## Flow

The script runs 10 test cases sequentially:

1. **Classic word count** — `ReadLines → Tokenize(lowercase) → Count → Sort(value, desc)`. Asserts "the" appears 6 times and "fox" appears 2 times.
2. **Top 5** — Same prefix but uses `TopN(5)` instead of `Sort`. Asserts exactly 5 results, ordered descending.
3. **Filter** — Counts, then keeps only words with count ≥ 2, then sorts by key. Asserts every result passes the predicate.
4. **Partition** — Counts, then partitions by first character. Asserts a file named `"t"` exists (for words starting with 't').
5. **External sort** — `Sort(by="key", memory_limit=3)` forces a merge-sort-style external sort. Asserts output is in sorted key order.
6. **Stats** — Reads `.stats` from the first pipeline. Asserts `total_records_in > 0` and that all 4 stages are tracked.
7. **Lazy evaluation** — `run_lazy()` returns an iterator. Asserts `next()` yields a value without consuming the full pipeline.
8. **FlatMap** — Splits a single `(index, line)` record into `(index, word)` pairs. Asserts transformation produces output.
9. **Empty input** — Pipeline with no input lines. Asserts the result is `[]`, not an error.
10. **File input** — `ReadLines` with a file path instead of a list. Asserts words from the file appear in the output.
11. **run_to_file** — Full pipeline writes JSONL output. Asserts the output file has content.

## Invariants

- **"the" appears exactly 6 times** across the three input lines — this is a manually-verified ground truth that anchors the word-count logic.
- **TopN preserves descending order**: `top5[0][1] >= top5[4][1]`.
- **Filter is a pure predicate**: after filtering for `count >= 2`, every record satisfies the predicate.
- **External sort produces the same result as in-memory sort**: output keys must be in sorted order regardless of `memory_limit`.
- **Empty input produces empty output**, not an error — the pipeline handles the zero-records case gracefully.
- **Stats track all stages**: `len(stats.stages) == 4` matches the number of `add_stage` calls on that pipeline.

## Error Handling

There is none — by design. This is a smoke test: if any assertion fails, execution halts with an `AssertionError` and the failing condition is printed in the f-string message. The `finally` blocks on file-based tests ensure cleanup even on failure, but there's no try/except for the pipeline operations themselves. The assumption is that the pipeline should never raise during normal operation; any exception is a test failure.

## Topics to Explore

- [file] `batch-word-count/pipeline.py` — The implementation of all pipeline stages and the `Pipeline` orchestrator; understanding how `run()` chains stage outputs, how `run_lazy()` yields lazily, and how external sort works with `memory_limit`
- [function] `batch-word-count/pipeline.py:Partition` — How records are routed to per-key output files, and the serialization format used
- [function] `batch-word-count/pipeline.py:Sort` — The external sort implementation when `memory_limit` is set — this is the most algorithmically interesting stage, modeling merge-sort from DDIA Ch. 10
- [file] `batch-word-count/tester_test_pipeline.py` — The meta-test wrapper that likely validates this test script itself or runs it in a controlled harness
- [general] `batch-dataflow-model` — How this pipeline maps to the batch processing concepts in DDIA Chapter 10 (MapReduce, dataflow engines, sort-merge joins)

## Beliefs

- `pipeline-records-are-tuples` — Pipeline stages produce and consume lists of `(key, value)` tuples, where key is a string and value depends on the stage (string for Tokenize, int for Count)
- `readlines-accepts-list-or-filepath` — `ReadLines` accepts either a `list[str]` of lines or a file path string, providing two input modes from the same stage
- `pipeline-run-returns-list` — `Pipeline.run()` returns a materialized `list` of all output records, while `run_lazy()` returns an iterator
- `external-sort-is-transparent` — `Sort` with `memory_limit` produces the same output as an in-memory sort — the external sort is an implementation optimization, not a semantic change
- `pipeline-stats-tracks-all-stages` — `Pipeline.stats.stages` contains one entry per `add_stage` call, and `total_records_in` reflects the count of records entering the pipeline

