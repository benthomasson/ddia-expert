# Function: merge_sstables in sstable-and-compaction/sstable.py

**Date:** 2026-05-28
**Time:** 19:01

# `merge_sstables` — K-Way Merge of Sorted String Tables

## Purpose

`merge_sstables` implements the core operation behind SSTable compaction: merging multiple sorted, immutable SSTable files into a single new SSTable. This is the fundamental building block that both size-tiered (STCS) and leveled (LCS) compaction strategies delegate to.

It exists because LSM-tree storage engines accumulate SSTables over time. Reads degrade as the number of SSTables grows (each must be checked), so compaction periodically merges them to reduce the count and reclaim space from obsolete or deleted entries.

## Contract

**Preconditions:**
- Each `SSTableReader` in `readers` must contain entries sorted by key (this is an SSTable invariant, not checked here).
- `output_path` must be a writable path; the file will be created or overwritten.
- Readers should not include the file at `output_path` (merging a file into itself would be undefined).

**Postconditions:**
- The output SSTable contains exactly one entry per unique key — the one with the highest timestamp across all inputs.
- Entries in the output are sorted by key.
- If `remove_tombstones=True`, entries where `value is None` (deletes) are dropped entirely.
- The input SSTables are **not** deleted or modified.

**Invariants during execution:**
- The min-heap always contains at most one entry per source SSTable.
- `last_key` tracks the most recently written key to deduplicate.

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `readers` | `List[SSTableReader]` | The SSTables to merge. Can be empty (produces an empty output). Order doesn't matter — the heap handles sorting. |
| `output_path` | `str` | Filesystem path for the merged SSTable. Parent directory must exist. |
| `remove_tombstones` | `bool` | When `True`, tombstones (deletion markers with `value=None`) are excluded from the output. Only safe during compaction when no older SSTables reference the same keys — otherwise the tombstone would be lost and the deleted key would "reappear." Defaults to `False`. |

## Return Value

Returns an `SSTableReader` opened on the newly written file. The caller can use it immediately for reads. The caller is responsible for:
- Tracking this reader in the `CompactionManager._sstables` list.
- Cleaning up old SSTable files if desired (this function does not delete inputs).

## Algorithm

The algorithm is a **k-way merge** using Python's `heapq` min-heap:

**1. Initialization** — Seed the heap with the first entry from each reader's `scan()` iterator. Each heap element is a 5-tuple:

```
(key, -timestamp, source_index, entry, iterator)
```

- `key` is the primary sort key (lexicographic, ascending).
- `-timestamp` is negated so that among entries with the same key, the **newest** sorts first (smallest negative = largest original timestamp).
- `source_index` is a tiebreaker to avoid comparing `SSTableEntry` objects (which aren't orderable). Without this, two entries with the same `(key, -timestamp)` would cause a `TypeError`.

**2. Main loop** — Pop the smallest element from the heap:

- **Duplicate key** (`key == last_key`): This entry has the same key as the one we just wrote but a lower timestamp (or same timestamp from a different source). Skip it — the winning entry was already emitted. Advance that source's iterator.
- **Tombstone removal** (`remove_tombstones and entry.value is None`): Skip deleted entries. Advance the iterator.
- **Normal case**: Write the entry to the output SSTable. Advance the iterator.

After processing each entry, `_push_next` pulls the next entry from that source's iterator into the heap, maintaining the invariant of one heap entry per active source.

**3. Finalization** — Call `writer.finish()` to write the sparse index and footer, then return a reader on the new file.

**Key insight**: Because each SSTable is already sorted by key, and the heap always yields the globally smallest `(key, -timestamp)`, we get a correct merge in O(N log K) time where N is total entries and K is the number of input SSTables.

## Side Effects

- **File I/O**: Creates and writes a new SSTable file at `output_path`. Each input SSTable is opened for reading via `scan()`.
- **No cleanup**: Input files are untouched. The caller (`CompactionManager._stcs_compact` and `_lcs_compact`) handles removing old readers from the active list, but notably does **not** delete the old files from disk either.

## Error Handling

This function does not catch or handle errors explicitly:

- If `output_path` is not writable, `SSTableWriter.__init__` raises `OSError`/`PermissionError`. The file may be left in a partial state.
- If a reader's file is corrupt or truncated, `_read_entry` inside `scan()` may return `None` early (truncating that source) or raise `struct.error` / `UnicodeDecodeError`.
- There is no atomicity guarantee — a crash mid-write leaves a partial output file. A production system would write to a temp file and rename.
- `StopIteration` from exhausted iterators is caught silently (correct behavior — an empty SSTable simply contributes nothing).

## Usage Patterns

Called exclusively by `CompactionManager`:

```python
# Size-tiered: merge a bucket of similarly-sized SSTables
merged = merge_sstables(bucket, self._next_path())

# Leveled: merge one SSTable with its overlapping neighbors in the next level
merged = merge_sstables(merge_set, self._next_path())
merged.level = lvl + 1
```

The caller always sets `.level` on the returned reader afterward (for leveled compaction). `merge_sstables` itself doesn't concern itself with levels.

Neither call site passes `remove_tombstones=True`, so tombstones are currently preserved through compaction. In a production system, tombstones would be removed during major compaction (when merging the oldest level) but retained otherwise to prevent ghost reads from older SSTables.

## Dependencies

- **`heapq`** (stdlib) — min-heap for k-way merge.
- **`SSTableWriter`** — writes the merged output in the binary SSTable format.
- **`SSTableReader`** — reads each input via `scan()` and wraps the output file.
- **`SSTableEntry`** — the dataclass flowing through the merge; must have `.key`, `.value`, `.timestamp`.

---

## Topics to Explore

- [function] `sstable-and-compaction/sstable.py:SSTableWriter.add` — Understand the on-disk binary format that `merge_sstables` writes into, including the sparse index and tombstone encoding
- [function] `sstable-and-compaction/sstable.py:CompactionManager._lcs_compact` — See how leveled compaction selects which SSTables to merge and how it maintains the level invariant (non-overlapping key ranges within a level)
- [function] `sstable-and-compaction/sstable.py:CompactionManager._stcs_compact` — Contrast size-tiered strategy: buckets SSTables by file size rather than by level/key-range overlap
- [file] `sstable-and-compaction/test_sstable.py` — Verify the merge semantics with concrete examples: duplicate keys, tombstone handling, empty inputs
- [general] `sstable-tombstone-gc-safety` — When is it safe to remove tombstones? Relates to DDIA's discussion of compaction and the risk of "resurrecting" deleted data if tombstones are dropped too early

## Beliefs

- `merge-dedup-by-newest-timestamp` — When multiple SSTables contain the same key, `merge_sstables` keeps only the entry with the highest timestamp; all older versions are silently discarded.
- `merge-preserves-tombstones-by-default` — Tombstones (`value=None`) are written to the output unless `remove_tombstones=True` is explicitly passed, and no current call site enables removal.
- `heap-tiebreak-prevents-entry-comparison` — The `source_index` field in the heap tuple exists solely to prevent Python from comparing `SSTableEntry` objects when `(key, -timestamp)` ties, which would raise `TypeError`.
- `merge-no-atomic-write` — The output file is written directly to `output_path` with no temp-file-and-rename pattern, so a crash mid-merge leaves a corrupt partial file.
- `merge-does-not-delete-inputs` — Input SSTable files are never deleted or modified by `merge_sstables`; cleanup is the caller's responsibility (and the current callers don't delete files either).

