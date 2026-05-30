# Topic: In real systems, LCS picks the SSTable with the *least* overlap to minimize per-compaction write amplification; this implementation picks the *most* — explore why and what the tradeoffs are

**Date:** 2026-05-29
**Time:** 12:37

# LCS SSTable Selection: Most-Overlap vs. Least-Overlap

## The Implementation's Choice

The critical line is at `sstable-and-compaction/sstable.py:425-429`:

```python
# Pick SSTable with most overlap with next level
best = max(level_tables,
           key=lambda s: len(self._overlapping(s, next_level)))
overlapping_next = self._overlapping(best, next_level)
merge_set = [best] + overlapping_next
```

The `_overlapping` helper (line 392) finds all SSTables in a target level whose key range intersects with a given SSTable. The selection strategy then picks the L0 SSTable that touches the **most** L1 SSTables, and merges them all together.

## How Production LCS Differs

In production systems like Cassandra or RocksDB, leveled compaction picks the SSTable with the **least** overlap with the next level. The reasoning is straightforward: each compaction must read *and rewrite* every overlapping SSTable in the target level. If an SSTable in L0 overlaps with 2 SSTables in L1, you rewrite 3 files. If it overlaps with 8, you rewrite 9. Picking the least overlap minimizes the bytes rewritten per compaction — the classic write amplification metric.

## Why This Implementation Inverts the Choice

This is a **pedagogical simplification** that prioritizes a different goal: **convergence speed toward a clean level structure**. Here's the reasoning:

1. **Maximizing cleanup per compaction round.** By picking the SSTable with the most overlap, a single `run_compaction()` call resolves the largest cluster of key-range conflicts between levels. The test at `sstable-and-compaction/test_sstable.py:108-121` creates just 3 L0 SSTables and expects one compaction call to promote them to L1. With a least-overlap strategy, you might need multiple rounds to clear L0 — each round surgically merging a small sliver.

2. **The teaching context doesn't have background compaction.** Production LSM engines run compaction continuously in background threads, so "many small compactions" is fine — each one is cheap. This implementation runs compaction as an explicit `run_compaction()` call (observable in both `test_sstable.py:47` for size-tiered and line 108 for leveled). When compaction is a discrete, user-triggered event rather than a continuous process, doing more work per invocation is pragmatic.

3. **No sustained write load to amortize over.** Write amplification matters when you're compacting while new writes keep arriving. In a test harness that writes a batch, compacts, then asserts, there's no concurrent write pressure making expensive compactions painful. The tradeoff that makes least-overlap essential in production (steady-state write throughput) simply doesn't apply here.

## The Tradeoff Matrix

| Dimension | Most-overlap (this impl) | Least-overlap (production) |
|-----------|-------------------------|---------------------------|
| Bytes rewritten per compaction | High — touches many L1 SSTables | Low — surgical, minimal rewrite |
| Compaction rounds to clear L0 | Fewer — resolves large overlap clusters | More — chips away incrementally |
| Write amplification (steady state) | Worse — rewrites more data per trigger | Better — minimizes per-op cost |
| Read amplification after compaction | Same end state — levels are sorted | Same end state |
| Implementation complexity | Lower — greedy, one-shot | Higher — needs round-robin or scoring |
| Suitability for background compaction | Poor — long pauses, bursty I/O | Good — predictable, small I/O bursts |

## The Deeper Point

The choice illuminates something important about compaction strategy design: the **selection heuristic only matters under sustained write load with background compaction**. In a batch-then-compact model (which is what this teaching implementation uses), the total bytes rewritten to reach a clean state is roughly the same regardless of selection order — you're merging the same data either way. The difference is whether you do it in one expensive pass or many cheap ones. Production systems care deeply about this because each pass competes with foreground writes for disk bandwidth. A teaching implementation doesn't have that constraint.

---

## Topics to Explore

- [function] `sstable-and-compaction/sstable.py:_overlapping` — The key-range intersection logic that determines merge sets; understanding its boundary conditions reveals how SSTables in a level can partially or fully overlap
- [general] `write-amplification-measurement` — Instrument both strategies to count total bytes written during a sequence of inserts + compactions; empirically verify that the total converges but the per-compaction distribution differs
- [function] `log-structured-merge-tree/lsm.py:compact` — The LSM module's simpler compaction (merges all SSTables into one) contrasts with the leveled strategy; compare the two to understand why levels exist at all
- [file] `sstable-and-compaction/sstable.py` — The `_size_tiered_compaction` method (the other strategy) picks SSTables by similar size rather than overlap — comparing the two strategies side-by-side clarifies when each is appropriate
- [general] `rocksdb-compaction-picker` — RocksDB's `LevelCompactionBuilder::PickFileToCompact` uses a round-robin pointer plus overlap scoring; studying it shows how production systems balance fairness and write amplification

## Beliefs

- `lcs-picks-most-overlap` — The leveled compaction implementation at `sstable.py:425-427` selects the L0 SSTable with the **most** next-level overlap via `max()`, inverting the production heuristic of least-overlap
- `overlapping-determines-merge-set` — The merge set for leveled compaction is always exactly one source SSTable plus all SSTables in the target level whose key range intersects it (`sstable.py:428-429`)
- `compaction-is-explicit-not-background` — Both the LSM and SSTable modules trigger compaction via explicit method calls (`compact()` / `run_compaction()`), not background threads — this removes the write-amplification pressure that motivates least-overlap selection in production
- `leveled-compaction-promotes-to-level-1` — After leveled compaction, the resulting SSTables are assigned `level = max_level + 1` (observable in the test assertion at `test_sstable.py:119`: `result[0].level == 1`), establishing the level hierarchy

