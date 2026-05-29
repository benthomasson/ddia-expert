# Function: CompactionManager in sstable-and-compaction/sstable.py

**Date:** 2026-05-29
**Time:** 08:58

# `CompactionManager`

## Purpose

`CompactionManager` is the orchestrator that decides **when** and **how** to merge SSTables together to reclaim space and bound read amplification. It implements two classic compaction strategies from Chapter 3 of DDIA:

- **Size-Tiered Compaction (STCS)**: groups SSTables into buckets by file size, merges when too many accumulate in one bucket. Write-optimized — good throughput, but can temporarily double space usage during compaction.
- **Leveled Compaction (LCS)**: organizes SSTables into levels with exponentially increasing size budgets, merges individual SSTables down into the next level by key-range overlap. Read-optimized — guarantees bounded read amplification at the cost of higher write amplification.

The class exists to decouple the compaction policy from the SSTable storage format itself (`SSTableWriter`/`SSTableReader`).

## Contract

**Preconditions:**
- `data_dir` must be a writable directory that already exists — the class never creates it.
- All `SSTableReader` instances added via `add_sstable` must have valid, readable backing files.
- For leveled compaction, each `SSTableReader.level` must be set correctly before registration. The manager trusts this attribute blindly.

**Postconditions:**
- After `run_compaction()`, the internal `_sstables` list reflects the post-merge state: input SSTables are removed, output SSTables are appended.
- The old SSTable files on disk are **not** deleted — only new merged files are created. Cleanup is the caller's responsibility.

**Invariants:**
- `_sstables` always contains the current set of live SSTables. There is no separate metadata store.
- Each call to `run_compaction()` performs at most **one** merge operation (one bucket for STCS, one level for LCS), then returns. The caller must loop if multiple rounds are needed.

## Parameters

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `data_dir` | `str` | required | Directory where merged SSTable files are written |
| `strategy` | `str` | `"size_tiered"` | Either `"size_tiered"` or anything else (treated as leveled) |
| `min_threshold` | `int` | `4` | STCS: minimum SSTables in a size bucket to trigger compaction |
| `size_thresholds` | `List[int]` | `[1M, 10M, 100M]` | STCS: byte boundaries between tiers — files <1MB are tier 0, <10MB tier 1, etc. |
| `l0_compaction_trigger` | `int` | `4` | LCS: number of L0 SSTables that triggers flush to L1 |
| `level_base_size` | `int` | `10_000_000` | LCS: max total bytes for level 1 |
| `fanout` | `int` | `10` | LCS: size multiplier between levels (`level_n_max = base * fanout^(n-1)`) |
| `max_levels` | `int` | `7` | LCS: highest level to consider |

**Edge case:** `strategy` is compared with `==` against `"size_tiered"` — any other string silently selects leveled compaction. There's no validation.

## Return Value

`run_compaction()` returns `List[SSTableReader]` — the newly created merged SSTables. Returns an empty list if no compaction was needed. The caller can use this to know what was produced, but the manager has already updated its internal state.

`needs_compaction()` returns `bool` — a pure check with no side effects.

## Algorithm

### Size-Tiered Compaction

1. **Bucket** each SSTable by file size into tiers using `_size_thresholds`. A 500KB file goes to tier 0, a 5MB file to tier 1, etc.
2. **Check** if any bucket has `>= min_threshold` SSTables.
3. **Merge** all SSTables in each qualifying bucket into a single new SSTable via `merge_sstables` (k-way merge by key, newest timestamp wins on duplicates).
4. **Remove** the inputs from `_sstables`, **append** the output. Note: STCS processes all qualifying buckets in one call, unlike LCS.

### Leveled Compaction

Two distinct paths:

**L0 → L1 compaction** (takes priority):
1. When `>= l0_trigger` SSTables accumulate in L0, collect them all.
2. Find every L1 SSTable whose key range overlaps with any L0 SSTable.
3. Merge the entire set (all L0 + overlapping L1) into a single SSTable assigned to level 1.

**Level N → N+1 compaction:**
1. Walk levels 1 through `max_levels - 1`. Find the first level where total file size exceeds `level_base_size * fanout^(level-1)`.
2. Pick the SSTable in that level with the **most** overlap with the next level (maximizing the number of overlapping SSTables — this is a heuristic to reduce future compaction work).
3. Merge that SSTable plus all overlapping SSTables from level N+1 into a single new SSTable assigned to level N+1.
4. Return immediately — only one level is compacted per call.

## Side Effects

- **File creation**: every merge writes a new `.sst` file to `data_dir` with a monotonic counter + millisecond timestamp in the filename.
- **State mutation**: `_sstables` is modified in-place during compaction — inputs removed, outputs appended.
- **Level assignment**: after merging, `merged.level` is set directly on the `SSTableReader` instance (mutating an attribute that the reader itself doesn't manage).
- **No file deletion**: old SSTable files remain on disk after their readers are removed from `_sstables`.

## Error Handling

There is essentially none. The class relies on:
- `merge_sstables` raising on I/O errors (which propagate uncaught).
- `list.remove()` raising `ValueError` if an SSTable isn't found — this would indicate a bug in the manager's bookkeeping.
- No protection against concurrent modification of `_sstables`.

## Usage Patterns

Typical usage in a write-ahead-log + memtable system:

```python
mgr = CompactionManager("/data/sst", strategy="size_tiered", min_threshold=4)

# After flushing a memtable to disk:
reader = SSTableReader("/data/sst/new_table.sst")
mgr.add_sstable(reader)

# Periodically or after flush:
while mgr.needs_compaction():
    new_tables = mgr.run_compaction()
    # Optionally: delete old files, update manifest, etc.
```

**Caller obligations:**
- Must call `needs_compaction()` or `run_compaction()` periodically — there's no background thread.
- Must manage SSTable file lifecycle (deletion of old files).
- For LCS, must set `.level` on SSTables before adding them.
- Must not add the same SSTable instance twice.

## Dependencies

| Dependency | Usage |
|-----------|-------|
| `merge_sstables` (same module) | K-way merge of multiple SSTables into one |
| `SSTableReader` (same module) | Read access to SSTable files, provides `metadata()`, `scan()`, and `.level` |
| `SSTableWriter` (via `merge_sstables`) | Creates the output SSTable |
| `os.path.join` | Path construction for output files |
| `time.time` | Timestamp in output filenames (collision avoidance) |

## Topics to Explore

- [function] `sstable-and-compaction/sstable.py:merge_sstables` — The k-way merge that does the actual work; understanding its deduplication and tombstone handling is critical to understanding what compaction achieves
- [file] `sstable-and-compaction/test_sstable.py` — See how STCS and LCS are exercised, including edge cases around overlapping key ranges and multi-level cascades
- [function] `sstable-and-compaction/sstable.py:SSTableReader.metadata` — The metadata contract that compaction depends on for sizing and key-range overlap decisions
- [general] `stcs-vs-lcs-tradeoffs` — DDIA Chapter 3 covers why STCS favors write throughput while LCS bounds read amplification, and when each is appropriate (Cassandra uses STCS by default, RocksDB uses leveled)
- [general] `compaction-output-splitting` — Production systems split merge output into multiple fixed-size SSTables rather than one giant file; this implementation merges into a single file, which limits LCS correctness (L1+ should have non-overlapping key ranges across files)

## Beliefs

- `stcs-merges-all-qualifying-buckets-per-call` — Size-tiered `run_compaction()` processes every bucket that meets `min_threshold` in a single call, while leveled compaction processes at most one level per call
- `lcs-produces-single-output-sstable` — Leveled compaction merges all inputs into one SSTable rather than splitting output by size, which means levels above L0 will contain overlapping key ranges after compaction — violating the LCS invariant that each level has non-overlapping SSTables
- `compaction-never-deletes-old-files` — Old SSTable files remain on disk after compaction; only the in-memory `_sstables` list is updated
- `strategy-selection-is-binary` — Any strategy string other than `"size_tiered"` silently selects leveled compaction with no validation or error
- `lcs-picks-max-overlap-sstable` — For level N→N+1 compaction, the SSTable with the most overlapping neighbors in the next level is chosen, which is a heuristic to consolidate fragmented key ranges

