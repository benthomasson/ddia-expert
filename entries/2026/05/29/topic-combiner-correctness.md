# Topic: Combiners must be associative and commutative to be correct (e.g., sum works, average doesn't). The framework doesn't enforce this — worth understanding why and what breaks if violated

**Date:** 2026-05-29
**Time:** 13:33

# Combiners Must Be Associative and Commutative

## The Core Idea

A combiner is a **local pre-aggregation** that runs on each mapper's output before the shuffle phase. It reduces network/disk traffic by partially reducing data at the source. The critical contract: **adding a combiner must not change the final result** — it's purely an optimization.

This only holds if the combiner function is associative and commutative. The framework accepts any callable and trusts you to get this right.

## How the Combiner Runs in This Code

Look at `mapreduce-framework/mapreduce.py:104-115`. Inside `_run_mapper`, after the map phase produces key-value pairs partitioned by reducer, the combiner runs **per-partition, per-mapper**:

```python
# Apply combiner if present
if self.combiner:
    for p in partitions:
        partitions[p] = self._apply_combiner(partitions[p])
```

Then `_apply_combiner` (line 123-130) groups by key and calls the combiner function on each group:

```python
def _apply_combiner(self, pairs):
    pairs.sort(key=lambda x: x[0])
    combined = []
    for key, group in groupby(pairs, key=lambda x: x[0]):
        values = [v for _, v in group]
        combined.extend(self.combiner(key, values))
    return combined
```

The reducer later sees the combiner's output **merged with outputs from other mappers** (line 137-148 in `_run_reducer`). It groups by key again and reduces, treating the combiner's output exactly like raw mapper output.

## Why Associativity and Commutativity Matter

The combiner sees an **arbitrary subset** of values for a key — only those produced by one mapper's chunk. The reducer then aggregates across all mappers. This means:

1. **Associativity** is required because the operation is applied in two stages: `reduce(combine(chunk1), combine(chunk2))` must equal `reduce(chunk1 + chunk2)`. Sum satisfies this: `sum([sum([1,2]), sum([3])]) == sum([1,2,3])`. Average does not: `avg([avg([1,2]), avg([3])]) = avg([1.5, 3]) = 2.25 ≠ avg([1,2,3]) = 2.0`.

2. **Commutativity** is required because the framework makes no guarantees about which mapper processes which records. The input split depends on `num_mappers` (line 86-96, `_split_input`), and the partitioning uses `hash(k) % self.num_reducers` (line 109). Different worker counts produce different groupings — the test at line 43-52 (`test_multi_worker_consistency`) explicitly verifies this doesn't affect results.

## What the Test Proves (and Doesn't)

The combiner test at `mapreduce-framework/test_mapreduce.py:54-68` uses `word_count_reducer` (which sums — an associative, commutative operation) as the combiner and asserts `r1 == r2`: same result with and without the combiner.

But notice what it **doesn't** test: there's no negative test showing a broken combiner producing wrong results. The framework's type signature at line 32 is:

```python
combiner: Optional[Callable[[Any, list[Any]], list[tuple[Any, Any]]]] = None
```

It accepts any callable with the right shape. No runtime check for mathematical properties. If you passed an averaging combiner, the code would run without errors but produce silently wrong results that vary with `num_mappers`.

## A Concrete Failure Scenario

Imagine computing average word length with a combiner that averages values:

- Mapper 1 emits: `("a", 3), ("a", 5)` → combiner produces `("a", 4.0)`
- Mapper 2 emits: `("a", 7)` → combiner produces `("a", 7.0)`
- Reducer sees `[4.0, 7.0]`, averages to `5.5`
- Correct answer: `avg(3, 5, 7) = 5.0`

The result changes depending on how input splits across mappers — a bug that's nondeterministic in production and invisible in small-scale tests.

## The Batch Pipeline's Approach

The `batch-word-count/pipeline.py` takes a different design path. Its `Count` stage (line 100-106) does full in-memory aggregation in a single pass — no combiner concept at all. This sidesteps the correctness trap entirely but gives up the distributed pre-aggregation benefit that combiners provide in a real multi-node MapReduce.

---

## Topics to Explore

- [function] `mapreduce-framework/mapreduce.py:_run_reducer` — See how combiner output merges with raw mapper output in the shuffle phase, and why the reducer can't distinguish between the two
- [function] `mapreduce-framework/mapreduce.py:_split_input` — Understand how input partitioning varies with worker count, which is why combiner correctness can't depend on which records land together
- [file] `batch-word-count/pipeline.py` — Compare the pipeline's single-pass `Count` stage with MapReduce's two-phase combine-then-reduce approach
- [general] `combiner-as-reducer-reuse` — The test uses `word_count_reducer` as the combiner (line 62), a common pattern; explore when the reducer can safely double as a combiner and when it can't
- [function] `mapreduce-framework/test_mapreduce.py:test_multi_worker_consistency` — Shows that correct operations produce identical results across worker configurations — the property a broken combiner would violate

## Beliefs

- `combiner-runs-per-mapper-partition` — The combiner applies independently to each mapper's partition output (`_run_mapper` line 113-115), not to the global key space, so it sees an arbitrary subset of values for any given key
- `combiner-not-type-checked` — The combiner's callable signature (`Callable[[Any, list[Any]], list[tuple[Any, Any]]]`) matches the reducer's, but no runtime or static check verifies associativity or commutativity
- `combiner-does-not-affect-map-output-stats` — `map_output_records` is incremented before the combiner runs (line 108), so stats reflect pre-combiner volume; the test at line 68 asserts this explicitly
- `combiner-output-indistinguishable-from-mapper-output` — The reducer reads combiner output from intermediate JSON files identically to raw mapper output, with no metadata distinguishing the two

