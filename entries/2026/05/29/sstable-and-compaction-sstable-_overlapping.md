# Function: _overlapping in sstable-and-compaction/sstable.py

**Date:** 2026-05-29
**Time:** 12:36

# `CompactionManager._overlapping`

## Purpose

This method finds which SSTables from a candidate set have key ranges that overlap with a given SSTable. It's the core spatial query behind leveled compaction — when compacting an SSTable from level N into level N+1, you need to know which level N+1 SSTables share key space with it, because those are the ones that must participate in the merge to maintain the leveled compaction invariant (no overlapping key ranges within a single level).

## Contract

- **Precondition**: All SSTable readers must be valid and readable (their metadata is derived at construction time, so no I/O happens here). Each SSTable's `min_key <= max_key` — this is guaranteed by `SSTableWriter.add` requiring sorted insertion order.
- **Postcondition**: Returns exactly the subset of `candidates` whose key ranges overlap with `sst`. The returned list preserves the order from `candidates`. The input lists are not modified.
- **Invariant**: The overlap test is symmetric — if A overlaps B, then B overlaps A. This method only tests one direction (candidates against sst), but the underlying math is commutative.

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `sst` | `SSTableReader` | The SSTable whose key range defines the "query window." In practice, this is the SSTable being promoted from level N. |
| `candidates` | `List[SSTableReader]` | SSTables to test against. In practice, these are all SSTables at the target level (N+1). |

**Edge cases**: If `candidates` is empty, returns `[]`. If `sst` contains a single key (min_key == max_key), the test still works correctly — it finds any candidate whose range contains that key.

## Return Value

A `List[SSTableReader]` — the subset of `candidates` that overlap. Can be empty (no overlap), or equal to the entire `candidates` list (full overlap). The caller uses this to build a merge set for compaction (`_lcs_compact`).

## Algorithm

The overlap check uses the standard interval overlap test. Two ranges `[A_min, A_max]` and `[B_min, B_max]` overlap if and only if `A_min <= B_max AND A_max >= B_min`. Equivalently, they do *not* overlap when one is entirely before or after the other.

Step by step:

1. Fetch `sst`'s metadata once (avoiding repeated calls).
2. For each candidate `c`, fetch its metadata and test: `c.min_key <= sst.max_key AND c.max_key >= sst.min_key`.
3. Collect all candidates that pass the test.

The comparison uses Python's default string ordering (lexicographic/byte-order), which matches the sorted order used by `SSTableWriter.add`.

## Side Effects

None. This is a pure query — no I/O, no mutation, no state changes. `metadata()` constructs a new `SSTableMetadata` dataclass from cached fields on each call, but that's allocation-only.

## Error Handling

No explicit error handling. If a reader is in a bad state (e.g., corrupted file that failed at construction), the error would have been raised during `SSTableReader.__init__`, not here. The method assumes all readers were constructed successfully.

## Usage Patterns

Called in two contexts within `_lcs_compact`:

1. **L0 compaction**: For each L0 SSTable, find which L1 SSTables overlap. L0 SSTables can have overlapping ranges with each other (that's why L0 exists), so this is called once per L0 SSTable.

2. **Level N compaction**: First used to *pick* which SSTable to compact (the one with the most overlapping SSTables in level N+1 — a heuristic to reduce future compaction work). Then called again to get the actual merge set.

```python
# Picking the best candidate to compact
best = max(level_ssts, key=lambda s: len(self._overlapping(s, next_level)))
# Getting its merge partners
overlapping_next = self._overlapping(best, next_level)
```

Note the double call for `best` — once inside `max` and once after. This is O(n*m) where n = level size, m = next level size.

## Dependencies

- `SSTableReader.metadata()` — returns an `SSTableMetadata` dataclass with `min_key` and `max_key` fields.
- Python's string comparison (`<=`, `>=`) — the correctness depends on keys being compared in the same order they were sorted when written.

## Assumptions Not Enforced by Types

1. **Key ordering is consistent**: The method assumes lexicographic string comparison matches the sort order used when building SSTables. If keys were sorted with a custom comparator, this overlap test would give wrong results.
2. **Metadata reflects actual content**: `min_key` and `max_key` are derived during `SSTableReader.__init__` by scanning entries. If the file were truncated or corrupted after the reader was constructed (e.g., by a concurrent process), the metadata would be stale — but this is an unusual failure mode.
3. **Keys are non-empty**: The `SSTableReader` defaults `min_key` and `max_key` to `""` for empty SSTables. An empty SSTable passed as `sst` would match any candidate whose `min_key` is `""`, which could produce spurious overlaps. In practice this doesn't happen because empty SSTables aren't fed into compaction.

## Topics to Explore

- [function] `sstable-and-compaction/sstable.py:_lcs_compact` — The caller that uses overlap results to build merge sets and enforce the leveled compaction invariant
- [function] `sstable-and-compaction/sstable.py:merge_sstables` — The k-way merge that consumes the overlap set and produces a single compacted SSTable
- [function] `sstable-and-compaction/sstable.py:_stcs_compact` — Size-tiered compaction for contrast — groups by file size tiers instead of key-range overlap
- [general] `leveled-compaction-write-amplification` — Why leveled compaction trades higher write amplification for bounded read amplification and space amplification
- [file] `sstable-and-compaction/test_sstable.py` — Test cases that exercise the compaction paths and verify overlap logic

## Beliefs

- `overlapping-uses-interval-overlap-test` — `_overlapping` uses the standard two-condition interval overlap test (`a.min <= b.max AND a.max >= b.min`) on SSTable key ranges using lexicographic string comparison
- `overlapping-is-pure-query` — `_overlapping` performs no I/O, mutation, or side effects; it operates entirely on in-memory metadata cached during SSTableReader construction
- `lcs-compact-calls-overlapping-twice-for-level-n` — During level N compaction, `_overlapping` is called O(n) times to pick the best SSTable, then once more to get its merge set, making the selection O(n*m) in level sizes
- `empty-sstable-overlap-edge-case` — An empty SSTable (min_key="" and max_key="") passed to `_overlapping` could produce spurious matches against candidates with empty-string min_keys, but compaction never feeds empty SSTables

