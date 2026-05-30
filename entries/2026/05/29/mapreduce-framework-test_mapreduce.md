# File: mapreduce-framework/test_mapreduce.py

**Date:** 2026-05-29
**Time:** 13:31

## Purpose

This is the test suite for a MapReduce framework implementation — one of the DDIA reference modules. It validates that `MapReduceJob` and `MapReducePipeline` behave correctly across the full spectrum of MapReduce semantics: basic map/reduce, partitioning determinism, combiners, file I/O, fault tolerance, multi-stage pipelines, and job statistics. It serves as both a correctness harness and a living spec for the framework's API contract.

## Key Components

### Mapper/Reducer Functions (module-level)

**`word_count_mapper(key, value)`** — The canonical MapReduce example. Takes a line of text and emits `(word, 1)` pairs. Lowercases input, splits on whitespace. Used by most tests as the default mapper.

**`word_count_reducer(key, values)`** — Sums all values for a given key. Returns a single `(key, total)` pair. Also doubles as the combiner in `test_combiner`.

### Test Functions

| Test | What it validates |
|------|-------------------|
| `test_word_count` | Core map-shuffle-reduce on 3 input lines; asserts exact counts and total unique keys (8) |
| `test_multi_worker_consistency` | Determinism — results are identical across 4 different `(num_mappers, num_reducers)` configurations |
| `test_combiner` | Combiner produces identical final results; intermediate record counts match (combiner is applied post-counting) |
| `test_file_input` | `job.run()` accepts a file path string, not just in-memory pairs |
| `test_empty_input` | Edge case — empty list yields empty output and zero stats |
| `test_single_key` | All mapper output collapses to one key; reducer sees the full value set |
| `test_fault_tolerant_mode` | `fault_tolerant=True` silently skips records where the mapper raises |
| `test_strict_mode` | Default mode propagates mapper exceptions to the caller |
| `test_pipeline` | `MapReducePipeline` chains two stages; output of stage 1 feeds into stage 2 |
| `test_statistics` | `job.stats` tracks input/output record counts, worker counts, and elapsed time |

## Patterns

**Self-contained test runner.** The `if __name__ == '__main__'` block implements a minimal test harness — iterates over a list of test functions, catches exceptions, prints PASS/FAIL, and exits with appropriate status. This avoids a pytest dependency while still being pytest-compatible (all tests are module-level `test_*` functions).

**Fixture-free design.** Each test constructs its own `MapReduceJob` with inline mapper/reducer lambdas or local functions. No shared state between tests, so they're safe to run in any order.

**Parameterized-by-hand.** `test_multi_worker_consistency` manually loops over worker configurations rather than using parameterized test fixtures — keeps the test self-explanatory.

**Cleanup via try/finally.** `test_file_input` uses `tempfile.NamedTemporaryFile` with `delete=False` and explicitly `os.unlink`s in a `finally` block, ensuring cleanup even on assertion failure.

## Dependencies

**Imports:**
- `os`, `tempfile` — file I/O for `test_file_input`
- `sys` — `sys.exit()` in the manual runner
- `mapreduce.MapReduceJob` — the core job abstraction under test
- `mapreduce.MapReducePipeline` — multi-stage chaining

**Imported by:** Nothing — this is a leaf test module.

## Flow

Each test follows the same pattern:

1. Define mapper/reducer (or reuse the module-level word-count pair)
2. Construct a `MapReduceJob` with configuration (`num_mappers`, `num_reducers`, optional `combiner`, `fault_tolerant`)
3. Call `job.run(input_data)` where `input_data` is either a list of `(key, value)` tuples or a file path string
4. Assert on the returned list of `(key, value)` result pairs and/or `job.stats`

The pipeline test (`test_pipeline`) adds a second stage where the mapper swaps key/value from stage 1's output, creating an inverted index from count→words.

## Invariants

- **Deterministic output.** `test_multi_worker_consistency` asserts that varying `num_mappers` and `num_reducers` never changes the result set — the shuffle/partition step must be deterministic for a given input.
- **Combiner transparency.** The combiner must be semantically invisible — `test_combiner` asserts `r1 == r2` regardless of combiner presence.
- **Stats accuracy.** `test_statistics` hardcodes expected record counts (3 input, 11 intermediate, 8 output for the fox/dog corpus), so any change to the shuffle or dedup logic will break this test.
- **Strict mode is the default.** `test_strict_mode` constructs a job with no `fault_tolerant` flag and expects exceptions to propagate.

## Error Handling

Two modes are tested:

1. **Strict (default):** `test_strict_mode` expects `RuntimeError` to propagate out of `job.run()`. The test uses a manual try/except with `assert False` as the fallthrough — a pattern that ensures the test fails if no exception is raised.

2. **Fault-tolerant:** `test_fault_tolerant_mode` sets `fault_tolerant=True` and verifies that a `ValueError` from the mapper on key=2 is swallowed — the bad record is skipped, and only the good records appear in output.

The test file itself does not catch or wrap errors beyond these two tests; all other tests expect clean execution or will fail with an unhandled assertion error.

## Topics to Explore

- [file] `mapreduce-framework/mapreduce.py` — The implementation under test; shows how shuffle/partition, combiners, stats tracking, and fault tolerance actually work
- [function] `mapreduce-framework/mapreduce.py:MapReduceJob.run` — The core execution engine; understanding it explains why worker count doesn't affect results
- [function] `mapreduce-framework/mapreduce.py:MapReducePipeline.run` — How stage outputs are wired to stage inputs, and how `stage_stats` is accumulated
- [general] `combiner-vs-reducer-semantics` — Why the combiner must be associative/commutative and why `map_output_records` is counted before the combiner runs
- [file] `batch-word-count/pipeline.py` — A related batch processing implementation; compare its pipeline abstraction with `MapReducePipeline`

## Beliefs

- `mapreduce-results-deterministic` — `MapReduceJob.run()` produces identical output regardless of `num_mappers`/`num_reducers` configuration for the same input data
- `mapreduce-combiner-transparent` — Applying a combiner never changes final results; it only reduces intermediate record volume
- `mapreduce-strict-mode-default` — `MapReduceJob` defaults to strict mode where mapper/reducer exceptions propagate to the caller
- `mapreduce-run-accepts-filepath` — `MapReduceJob.run()` accepts either a list of `(key, value)` tuples or a file path string as input
- `mapreduce-stats-tracks-records` — `job.stats` accurately counts `map_input_records`, `map_output_records`, `reduce_output_records`, worker counts, and elapsed time after each run

