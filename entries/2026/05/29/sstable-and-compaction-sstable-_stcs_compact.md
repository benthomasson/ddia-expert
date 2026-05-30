# Function: _stcs_compact in sstable-and-compaction/sstable.py

**Date:** 2026-05-29
**Time:** 12:35

## `_stcs_compact` — Size-Tiered Compaction Strategy (STCS)

### Purpose

This is the core compaction routine for the size-tiered compaction strategy. It finds groups of similarly-sized SSTables that have accumulated past a threshold, merges each group into a single larger SSTable, and updates the manager's SSTable list. The goal: reduce the number of SSTables the system must search during reads, while grouping files by size so that small, recently-flushed tables merge with peers rather than being immediately folded into a massive file.

### Contract

**Preconditions:**
- `self._sstables` contains valid `SSTableReader` instances with readable files on disk.
- `self._data_dir` exists and is writable (needed by `_next_path()` for output files).
- The caller should have checked `_stcs_needs_compaction()` first, though calling this when no compaction is needed is safe — it returns an empty list and is a no-op.

**Postconditions:**
- Every bucket that met the threshold has been replaced: its individual SSTables are removed from `self._sstables` and one merged SSTable takes their place.
- Buckets below the threshold are untouched.
- The returned list contains exactly the newly created merged SSTables.

**Invariant:**
- Every key-value pair present in the input SSTables is present in the output (no data loss). The merge uses `merge_sstables`, which resolves duplicate keys by newest timestamp.

### Parameters

None beyond `self`. All configuration comes from instance state:

| Field | Role |
|-------|------|
| `self._sstables` | The live set of SSTables managed by this instance |
| `self._min_threshold` | Minimum bucket size to trigger a merge (default 4) |
| `self._size_thresholds` | Tier boundaries used by `_stcs_buckets()` → `_get_tier()` |

### Return Value

`List[SSTableReader]` — the newly created merged SSTables, one per compacted bucket. Empty list if no bucket met the threshold. The caller doesn't need to register these; the method already appends them to `self._sstables`.

### Algorithm

1. **Bucket SSTables by size tier.** Calls `_stcs_buckets()`, which groups every SSTable by its tier index (determined by comparing `file_size` against `self._size_thresholds`). Tier 0 is the smallest files, tier N is the largest.

2. **Iterate each bucket.** For each `(tier, bucket)` pair:
   - **Check threshold.** If the bucket has fewer than `_min_threshold` SSTables, skip it — not enough peers to justify a merge.
   - **Merge.** Call `merge_sstables(bucket, self._next_path())` — a k-way merge that produces a single sorted SSTable on disk. Duplicate keys resolve to the entry with the highest timestamp.
   - **Track the result.** Append the merged `SSTableReader` to `new_sstables`.
   - **Remove inputs.** Remove each original SSTable from `self._sstables` via `list.remove()`.

3. **Register new SSTables.** Extend `self._sstables` with the merged results.

4. **Return** the list of newly created SSTables.

### Side Effects

- **Disk I/O:** Creates one new `.sst` file per compacted bucket under `self._data_dir`.
- **Mutates `self._sstables`:** Removes compacted inputs, appends merged outputs.
- **Mutates `self._counter`:** Incremented by `_next_path()` for unique filenames.
- **Does NOT delete old SSTable files.** The original `.sst` files remain on disk even after their readers are removed from the managed list. This is a potential resource leak — the caller (or a higher-level component) is responsible for cleanup.

### Error Handling

Essentially none. Several failure modes propagate as unhandled exceptions:

- `list.remove(sst)` raises `ValueError` if an SSTable appears in multiple buckets or was already removed — shouldn't happen given `_get_tier` assigns each SSTable to exactly one tier, but the code doesn't defend against it.
- `merge_sstables` can raise I/O errors if the output path is unwritable or input files are corrupted.
- If `_next_path()` generates a path that collides with an existing file, `SSTableWriter` silently overwrites it (uses `open(..., "wb")`).

### Usage Patterns

Called exclusively through `run_compaction()` when the strategy is `"size_tiered"`:

```python
mgr = CompactionManager(data_dir, strategy="size_tiered", min_threshold=4)
# ... add SSTables over time ...
if mgr.needs_compaction():
    new = mgr.run_compaction()
```

Typically invoked after memtable flushes — each flush produces a small L0 SSTable, and once enough accumulate in the same size tier, they get merged. The merged result is larger, so it lands in a higher tier, where it eventually merges with other similarly-sized files. This creates a natural hierarchy of increasingly large SSTables over time.

### Dependencies

| Dependency | Role |
|------------|------|
| `_stcs_buckets()` / `_get_tier()` | Classify SSTables into size tiers |
| `merge_sstables()` | K-way sorted merge with timestamp-based deduplication |
| `_next_path()` | Generate unique output file paths |
| `SSTableReader` | Returned by `merge_sstables`, appended to the managed list |

### Assumptions Not Enforced by Types

1. **`self._sstables` contains no duplicates.** `list.remove()` removes the first match — if an SSTable appears twice, only one copy is removed, leaving the list inconsistent.
2. **SSTables are internally sorted.** `merge_sstables` assumes each input yields entries in key order. Nothing prevents adding an unsorted SSTable via `add_sstable()`.
3. **Old files are cleaned up externally.** The method orphans input files on disk.
4. **Concurrent access is unsafe.** The method mutates `self._sstables` while iterating buckets derived from it. Safe only because `_stcs_buckets()` creates a snapshot (new dict of lists), but concurrent calls to `add_sstable()` during compaction could corrupt state.

---

## Topics to Explore

- [function] `sstable-and-compaction/sstable.py:merge_sstables` — The k-way merge that powers both compaction strategies; understanding its dedup semantics (newest-timestamp-wins, tombstone handling) is essential to reasoning about compaction correctness.
- [function] `sstable-and-compaction/sstable.py:_lcs_compact` — Leveled compaction is the alternative strategy; compare its overlap-based promotion model with STCS's size-bucketing to understand the read-amplification vs write-amplification tradeoff.
- [function] `sstable-and-compaction/sstable.py:_get_tier` — The bucketing logic that determines which SSTables are peers; the threshold boundaries directly control how aggressively small files merge.
- [file] `sstable-and-compaction/test_sstable.py` — Tests reveal the expected compaction triggers and post-conditions that the implementation must satisfy.
- [general] `stcs-write-amplification` — STCS is optimized for write-heavy workloads but suffers from space amplification since it never removes tombstones during merge (note `remove_tombstones` defaults to `False`); worth understanding when this strategy is appropriate vs leveled.

---

## Beliefs

- `stcs-merges-within-size-tiers` — Size-tiered compaction only merges SSTables in the same size tier; it never merges a small SSTable directly with a much larger one.
- `stcs-min-threshold-default-four` — A size tier must accumulate at least 4 SSTables (the default `min_threshold`) before compaction merges them.
- `stcs-compact-preserves-tombstones` — `_stcs_compact` calls `merge_sstables` with `remove_tombstones=False` (the default), so deleted keys survive compaction as tombstone markers.
- `stcs-compact-does-not-delete-old-files` — After merging, the original SSTable files remain on disk; only the in-memory references in `self._sstables` are removed.
- `stcs-compact-mutates-sstable-list` — `_stcs_compact` both removes old and appends new entries to `self._sstables` as a side effect, so callers must not rely on the list being stable across a compaction call.

