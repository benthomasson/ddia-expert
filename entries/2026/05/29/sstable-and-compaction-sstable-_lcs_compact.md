# Function: _lcs_compact in sstable-and-compaction/sstable.py

**Date:** 2026-05-29
**Time:** 08:39

# `_lcs_compact` — Leveled Compaction Strategy

## Purpose

`_lcs_compact` executes one round of leveled compaction (LCS), the compaction strategy used by systems like LevelDB and RocksDB. It maintains the core LCS invariant: within each level (L1+), SSTable key ranges don't overlap, and each level has a size budget that grows exponentially. When a level exceeds its budget, one SSTable is pushed down to the next level by merging it with all overlapping SSTables there.

It exists as the counterpart to `_stcs_compact` (size-tiered), giving `CompactionManager` two pluggable compaction strategies behind a common interface.

## Contract

**Preconditions:**
- `self._sstables` contains valid `SSTableReader` instances with `.level` set correctly.
- L1+ SSTables within the same level have non-overlapping key ranges (the LCS invariant — assumed, not enforced).
- `self._strategy == "leveled"` (caller dispatches via `run_compaction`).

**Postconditions:**
- At most one compaction is performed per call. The method returns after the first level that needed work.
- `self._sstables` is mutated in place: the input SSTables in the merge set are removed and replaced by a single merged SSTable at the next level.
- The merged SSTable's `.level` is set to the target level.

**Invariant maintained:** After an L0 compaction, all L0 SSTables are consumed. After an L(N) compaction, the selected SSTable and its overlapping neighbors in L(N+1) are replaced by a single SSTable at L(N+1).

## Parameters

None beyond `self`. All configuration comes from instance attributes:

| Attribute | Role | Default |
|-----------|------|---------|
| `_l0_trigger` | Number of L0 SSTables that triggers L0→L1 compaction | 4 |
| `_max_levels` | Upper bound on level iteration (exclusive) | 7 |
| `_level_base_size` | Size budget for L1 in bytes | 10 MB |
| `_fanout` | Multiplier per level (L(N) budget = base × fanout^(N-1)) | 10 |

## Return Value

`List[SSTableReader]` — the newly created SSTables from this compaction round.

- **One element list**: a compaction happened; the single merged SSTable is the result.
- **Empty list**: no level needed compaction. The caller should interpret this as "nothing to do."

The caller (`run_compaction`) returns this directly. No error signaling via return value.

## Algorithm

The method has two distinct phases, tried in order:

### Phase 1: L0 Compaction

L0 is special — SSTables there can have overlapping key ranges (they come straight from memtable flushes).

1. Check if `len(L0) >= l0_trigger`. If not, skip to Phase 2.
2. Collect **all** L0 SSTables — every one participates.
3. For each L0 SSTable, find all L1 SSTables whose key ranges overlap with it (via `_overlapping`). Accumulate these by `id()` into a set to deduplicate (one L1 SSTable may overlap multiple L0 SSTables).
4. Build the merge set: all of L0 + the overlapping subset of L1.
5. K-way merge them into a single new SSTable, assign it `level = 1`.
6. Remove all input SSTables from `self._sstables`, add the merged one.
7. Return immediately — only one compaction per call.

### Phase 2: Level N Compaction (L1 through L_max-1)

Iterate levels bottom-up:

1. Sum the file sizes of all SSTables at this level.
2. Compare against `_level_max_size(lvl)` (= `base × fanout^(lvl-1)`).
3. If over budget, pick the SSTable with the **most overlap** with the next level. This is a heuristic — by picking the most-overlapping SSTable, the compaction rewrites the maximum amount of data that's already "in the way," reducing future compaction I/O.
4. Find all SSTables in `lvl+1` that overlap with the chosen one.
5. Merge them into a single SSTable at `lvl+1`.
6. Remove inputs, add output, return immediately.

If no level is over budget, return `[]`.

## Side Effects

- **Mutates `self._sstables`**: removes consumed SSTables, appends the merged one.
- **Disk I/O**: `merge_sstables` writes a new SSTable file. Old files are **not** deleted from disk — only removed from the in-memory list. This is a potential resource leak if the caller doesn't handle cleanup.
- **`self._counter` incremented** via `_next_path()` for unique file naming.
- **Sets `.level` on the new `SSTableReader`** — mutating a field that isn't part of the constructor.

## Error Handling

Essentially none. The method assumes:

- `self._sstables` is consistent (no duplicates, all readers valid).
- `merge_sstables` succeeds. A disk-full or I/O error would propagate as an unhandled exception, leaving `self._sstables` in a partially modified state (some SSTables already removed but no merged one added).
- `max()` on `level_ssts` won't receive an empty sequence — this is safe because the `total > threshold` check implies at least one SSTable exists, but worth noting.

## Usage Patterns

```python
manager = CompactionManager(data_dir, strategy="leveled")
manager.add_sstable(flushed_table)

while manager.needs_compaction():
    new_tables = manager.run_compaction()  # dispatches to _lcs_compact
```

The caller is expected to call `run_compaction` in a loop because each invocation handles at most one level. If L0 triggers compaction and the result pushes L1 over budget, a second call is needed to cascade L1→L2.

## Dependencies

| Dependency | Purpose |
|------------|---------|
| `merge_sstables()` | K-way merge of multiple SSTables into one, deduplicating by key (newest timestamp wins) |
| `SSTableReader` | The `.level` attribute, `.metadata()` for key ranges and file sizes, `.scan()` used internally by merge |
| `_overlapping()` | Key-range intersection test between one SSTable and a list of candidates |
| `_get_levels()` | Groups `self._sstables` by `.level` into a dict |
| `_level_max_size()` | Computes the size budget for a given level |
| `_next_path()` | Generates a unique file path for the compaction output |

---

## Topics to Explore

- [function] `sstable-and-compaction/sstable.py:merge_sstables` — The k-way merge that powers both compaction strategies; understanding its deduplication and tombstone semantics is essential to reasoning about compaction correctness
- [function] `sstable-and-compaction/sstable.py:_stcs_compact` — Compare size-tiered vs leveled compaction: STCS groups by size tier and merges entire buckets, while LCS maintains sorted runs per level — the tradeoff between write amplification and read amplification
- [function] `sstable-and-compaction/sstable.py:_overlapping` — The key-range overlap check that determines which SSTables get pulled into a compaction; its correctness is critical to maintaining the non-overlapping invariant at L1+
- [general] `leveled-compaction-write-amplification` — In real systems, LCS picks the SSTable with the *least* overlap to minimize per-compaction write amplification; this implementation picks the *most* — explore why and what the tradeoffs are
- [file] `sstable-and-compaction/test_sstable.py` — Verify how the test suite exercises edge cases like empty levels, cascading compactions, and the L0 trigger threshold

## Beliefs

- `lcs-compact-one-per-call` — `_lcs_compact` performs at most one merge operation per invocation; cascading compactions across levels require repeated calls
- `lcs-l0-compacts-all` — L0 compaction always includes every L0 SSTable in the merge set, not just a subset, which can cause large write spikes when `l0_trigger` is high
- `lcs-picks-most-overlap` — The level-N compaction heuristic picks the SSTable with the most overlap with the next level, which is the opposite of LevelDB/RocksDB's least-overlap heuristic and increases write amplification
- `lcs-no-old-file-cleanup` — Compacted SSTables are removed from `self._sstables` but their underlying files on disk are never deleted, creating a resource leak
- `lcs-level-size-exponential` — Level size budgets grow as `base_size × fanout^(level-1)`, so with defaults of 10MB base and 10× fanout, L1=10MB, L2=100MB, L3=1GB, etc.

