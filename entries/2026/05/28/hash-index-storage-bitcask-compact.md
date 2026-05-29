# Function: compact in hash-index-storage/bitcask.py

**Date:** 2026-05-28
**Time:** 18:22

# `BitcaskStore.compact`

## Purpose

`compact` is the garbage collection mechanism for a Bitcask append-only storage engine. Because Bitcask never overwrites data in place — every `put` and `delete` appends a new record — old data files accumulate stale entries (superseded values) and tombstones (deleted keys). `compact` reclaims disk space by rewriting only the live records from immutable (non-active) data files into fresh files, then deleting the originals.

This directly implements the "compaction" or "merging" process described in DDIA Chapter 3's discussion of hash-indexed log-structured storage.

## Contract

**Preconditions:**
- The store is open (active file handle is valid).
- No concurrent readers/writers — this implementation is **not thread-safe**. There are no locks, and the method mutates `keydir`, `file_handles`, `active_file_id`, and the filesystem simultaneously.

**Postconditions:**
- All immutable data files that existed before the call are deleted.
- A new set of merged files contains exactly the live records that were in those immutable files.
- The `keydir` is updated so all entries point to either the new merged files or the (renumbered) active file.
- Hint files are written alongside each merged file for fast future index rebuilds.
- The active file is renumbered to a file ID above the merged files and reopened.

**Invariants preserved:**
- File IDs remain monotonically increasing: merged files get IDs starting at `active_file_id + 1`, and the active file is renumbered to `merged_file_id + 1`.
- The `keydir` remains a consistent index: every key maps to the correct (file_id, offset, size) for its most recent live value.

## Parameters

None — operates entirely on `self` state.

## Return Value

`None`. All effects are side effects.

## Algorithm

The method proceeds in six phases:

### Phase 1: Identify immutable files
```python
immutable_ids = [fid for fid in all_ids if fid != self.active_file_id]
```
The active file is excluded because it's still being written to. If there are no immutable files, the method returns immediately.

### Phase 2: Scan immutable files for latest records
Iterates through each immutable file in ID order (oldest first), parsing every record by reading the binary header (`timestamp`, `key_size`, `value_size`) followed by key and value bytes. Builds a `latest` dict keyed by string key:

- **Tombstone** (`val_size == 0`): Removes the key from `latest` — the delete should propagate.
- **Live record**: Keeps the entry if it's newer (by timestamp) than any previously seen entry for that key, or if the key hasn't been seen yet.

Processing files in ascending ID order means later (newer) files naturally overwrite earlier entries, but the explicit timestamp comparison handles out-of-order cases.

### Phase 3: Filter against the keydir
```python
if entry and entry.file_id in immutable_ids:
    keys_to_compact[key] = (fid, offset, size, ts)
```
A key whose `keydir` entry points to the **active** file has a newer value there. The immutable copies are stale, so they're excluded from the merge output. This prevents compaction from resurrecting an old value that was overwritten in the active file.

### Phase 4: Write merged files
Closes readers for old immutable files, then writes each surviving record to new merged data files:

- New file IDs start at `active_file_id + 1` to avoid collisions.
- Records are re-read from old files (reopened fresh since the cached readers were closed), preserving the **original timestamp**.
- Merged files are rotated at `max_file_size`, just like active files.
- The `keydir` is updated in-place to point to the new locations.
- Hint file entries are accumulated and written per merged file for fast index rebuilds on restart.

### Phase 5: Delete old immutable files
Both `.data` and `.hint` files for every old immutable file ID are removed from disk.

### Phase 6: Renumber and reopen the active file
The active file must be renumbered so its ID is above the merged files (maintaining the monotonic ID invariant):

1. Closes the active file handle.
2. Renames the active data file from old ID to `merged_file_id + 1`.
3. Walks the `keydir` to update any entries that pointed to the old active file ID.
4. Reopens the active file at its new path.

## Side Effects

- **Filesystem**: Creates new `.data` and `.hint` files; deletes old `.data` and `.hint` files; renames the active data file.
- **In-memory state**: Mutates `self.keydir` (updates file_id/offset/size for compacted keys and active-file keys), `self.file_handles` (closes old, opens new), `self.active_file_id`, and `self.active_file`.
- **I/O**: Opens and closes multiple file handles during the process. Old immutable file readers are reopened individually (not reusing the cached handles that were closed earlier).

## Error Handling

There is essentially **none**. The method does not use `try/except` and makes no attempt at atomicity:

- A crash mid-compaction could leave the store in an inconsistent state: old files deleted but new files only partially written, or `keydir` entries pointing to deleted files.
- File I/O errors (permission denied, disk full) will propagate as unhandled exceptions, potentially leaving partial merged files on disk.
- No write-ahead log or two-phase commit protects the transition.

This is appropriate for a reference implementation but would be a serious concern in production.

## Usage Patterns

Compaction is triggered explicitly by the caller — there's no automatic background compaction:

```python
store = BitcaskStore("/tmp/data")
for i in range(100000):
    store.put(f"key-{i % 100}", f"value-{i}")
store.compact()  # reclaim space from ~99,900 stale entries
```

Callers should ensure no concurrent reads or writes during compaction. The method is idempotent in the sense that calling it when there are no immutable files is a no-op.

## Dependencies

- `os` — file operations (path, size, rename, remove, makedirs)
- `struct` — binary record parsing/packing using `HEADER_FORMAT` and `HINT_FORMAT`
- `KeyEntry` — dataclass for keydir entries
- Internal methods: `_find_file_ids`, `_data_path`, `_hint_path`, `_get_reader`, `_write_hint_file`, `_open_active_file`

## Unspoken Assumptions

1. **Single-writer exclusivity**: No other process or thread is reading or writing the data directory during compaction. This is not enforced by any locking mechanism.
2. **File IDs never wrap or collide**: The scheme of assigning `active_file_id + 1` as the base for merged files assumes there are no existing files at those IDs.
3. **Records fit in memory**: Each record is read entirely into memory. There is no streaming for large values.
4. **Timestamps are monotonically increasing**: The "latest wins" logic assumes timestamps from `time.time()` don't go backwards. Clock adjustments could violate this.
5. **The active file's data file still exists on disk at old_active_id**: The rename at the end assumes no external process has moved or deleted it.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:_rebuild_index` — How the keydir is reconstructed from data and hint files on startup, and why hint files make this fast
- [function] `hash-index-storage/bitcask.py:_write_hint_file` — The hint file format that compaction produces, enabling O(keys) index rebuilds instead of O(records)
- [file] `hash-index-storage/test_bitcask.py` — Tests that exercise compaction correctness: verify that stale entries are removed and live data survives
- [file] `sstable-and-compaction/test_sstable.py` — Compare Bitcask's hash-index compaction with SSTable/LSM-tree compaction strategies from the same DDIA chapter
- [general] `bitcask-crash-recovery` — How a production Bitcask (e.g., Riak's) handles crash-safety during compaction using atomic rename and CRC checksums

## Beliefs

- `bitcask-compact-preserves-timestamps` — Compaction rewrites records with their original timestamps, not the time of compaction, ensuring timestamp-based ordering is preserved across merges.
- `bitcask-compact-skips-active-file-keys` — Keys whose latest value is in the active file are excluded from compaction output; their immutable-file copies are treated as stale and discarded.
- `bitcask-compact-not-crash-safe` — Compaction is non-atomic: a crash between deleting old files and completing the rewrite can leave the store in an unrecoverable state with no rollback mechanism.
- `bitcask-compact-renumbers-active-file` — After compaction, the active file is renamed to a higher file ID to maintain the invariant that file IDs increase monotonically, and all keydir entries pointing to the old active ID are updated.
- `bitcask-compact-writes-hint-files` — Compaction produces hint files alongside each merged data file, allowing future index rebuilds to skip full record scanning.

