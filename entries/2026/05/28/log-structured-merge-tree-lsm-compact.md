# Function: compact in log-structured-merge-tree/lsm.py

**Date:** 2026-05-28
**Time:** 19:02

## `LSMTree.compact` — Full Compaction of SSTables

### Purpose

`compact` performs **full compaction** — it merges every SSTable on disk into a single new SSTable, deduplicating keys (newest value wins) and permanently removing tombstones (deleted keys). This reclaims disk space and reduces the number of files that `get` and `range_scan` must search.

In the DDIA model, this is a **size-tiered compaction** strategy at its simplest: all SSTables collapse into one. It's triggered automatically by `_flush` when `len(self._sstables) >= self._compaction_threshold`, or can be called directly.

### Contract

**Preconditions:**
- `self._sstables` is a list of `SSTable` objects sorted by `seq` (oldest first, newest last).
- Each SSTable file on disk is intact and readable.
- No concurrent writers are modifying the SSTable list (no thread safety).

**Postconditions:**
- `self._sstables` contains exactly one SSTable with a fresh sequence number.
- All old SSTable files are deleted from disk.
- The surviving SSTable contains every live key-value pair, with only the newest version of each key retained.
- All tombstones are purged — deleted keys disappear entirely.

**Invariant preserved:** SSTables remain sorted by sequence number (trivially — there's only one).

**Short-circuit:** If fewer than 2 SSTables exist, the method returns immediately — nothing to merge.

### Parameters

None beyond `self`. The method operates entirely on `self._sstables` and `self._dir`.

### Return Value

`None`. All effects are side effects.

### Algorithm

1. **Guard:** Exit early if there are 0 or 1 SSTables — compaction requires at least 2.

2. **Collect:** Iterate every SSTable via `scan_all()`, gathering `(key, seq, value)` triples. The `seq` comes from the SSTable object, not from individual entries — every entry in a given SSTable shares the same sequence number. This is a simplification: it means if SSTable 3 has keys `a, b, c`, all three get seq=3.

3. **Sort:** Sort the collected entries by `(key, -seq)`. This groups entries by key, with the highest sequence number (newest) first within each group.

4. **Deduplicate:** Walk the sorted list, keeping only the first occurrence of each key (the newest version). If that newest version is a `TOMBSTONE` (`b""`), skip it entirely — the key is dead.

5. **Write:** Create a new SSTable with a fresh sequence number. The `merged` list is already sorted by key (the sort in step 3 guarantees this), which is exactly what `SSTable.write` expects.

6. **Swap and clean up:** Replace `self._sstables` with a list containing only the new SSTable. Delete every old SSTable file from disk.

### Side Effects

- **Disk I/O (read):** Reads every entry from every SSTable file via `scan_all()`.
- **Disk I/O (write):** Creates one new SSTable file (`sst_{seq:06d}.sst`).
- **Disk I/O (delete):** Removes all previous SSTable files via `os.remove`.
- **State mutation:** Replaces `self._sstables` list. Increments `self._seq` via `_next_seq()`.
- **No WAL interaction:** Compaction doesn't touch the WAL. Immutable memtables and the active memtable are also untouched.

### Error Handling

There is **no error handling**. If any step fails:
- A crash during `scan_all()` leaves the system unchanged (the new file hasn't been written yet).
- A crash after `SSTable.write` but before old files are deleted leaves both old and new SSTables on disk. On restart, `_load_existing_sstables` would load all of them, creating duplicates — the data remains correct (newest-wins dedup still works at read time) but wastes space.
- A crash mid-deletion of old files leaves a partial set — same outcome as above, safe but wasteful.
- An `os.remove` failure (permissions, file locked) raises `OSError` and halts deletion partway through, leaving `self._sstables` pointing only at the new file while some old files remain as orphans.

### Usage Patterns

```python
# Automatically triggered inside _flush:
if len(self._sstables) >= self._compaction_threshold:
    self.compact()

# Or called directly by application code:
tree.compact()
```

Callers should not assume compaction is fast — it reads and rewrites the entire dataset. The default `compaction_threshold` of 4 means this runs every 4th flush.

### Dependencies

- `SSTable.scan_all()` — sequential read of all entries from an SSTable file.
- `SSTable.write()` — writes sorted entries to a new SSTable with sparse index.
- `os.path.join`, `os.remove` — filesystem operations.
- `TOMBSTONE` (`b""`) — the sentinel value for deleted keys.
- `self._next_seq()` — monotonic sequence number generator.

### Assumptions Not Enforced by Types

1. **`scan_all()` yields entries in sorted key order** — the sort in compact doesn't depend on this (it re-sorts everything), but `SSTable.write` requires sorted input, which is guaranteed by the `(key, -seq)` sort producing key-ascending order after dedup.
2. **Sequence numbers are unique per SSTable** — if two SSTables shared a `seq`, the sort would be nondeterministic for entries with the same key.
3. **No concurrent access** — another thread calling `get` during the swap between `self._sstables = [new_sst]` and the old file deletions would be fine, but a concurrent `_flush` appending to `self._sstables` could lose the appended SSTable.
4. **All values in `merged` are non-empty bytes** — the tombstone check ensures `b""` is never written, but there's no validation that other values are well-formed.
5. **Disk has space for the new SSTable before the old ones are deleted** — worst case, compaction temporarily doubles disk usage.

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:_flush` — The flush path that triggers compaction and manages the memtable-to-SSTable lifecycle
- [function] `log-structured-merge-tree/lsm.py:SSTable.write` — How the sorted entries and sparse index are serialized to the binary SSTable format
- [function] `log-structured-merge-tree/lsm.py:range_scan` — Uses priority-based merging across all layers (memtable + immutable + SSTables), a pattern compaction simplifies
- [general] `leveled-vs-size-tiered-compaction` — This implementation uses size-tiered (merge-all) compaction; DDIA Ch. 3 contrasts it with leveled compaction (LevelDB/RocksDB style) which bounds space amplification
- [file] `sstable-and-compaction/test_sstable.py` — Tests for SSTable compaction behavior that may cover edge cases like empty tables and tombstone-only merges

## Beliefs

- `compact-merges-all-sstables` — `compact` always merges every SSTable into one; there is no partial/leveled compaction strategy
- `compact-newest-wins-by-seq` — During compaction, the entry with the highest SSTable sequence number wins for each key; sequence is per-SSTable, not per-entry
- `compact-purges-tombstones` — Tombstones (`b""`) are permanently removed during compaction and never written to the output SSTable
- `compact-not-crash-safe` — A crash between writing the new SSTable and deleting all old ones leaves orphan files that `_load_existing_sstables` will reload (safe but wasteful)
- `compact-no-concurrency-safety` — The method mutates `self._sstables` without locking; concurrent `_flush` or `get` calls during compaction can produce incorrect state

