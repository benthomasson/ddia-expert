# Function: _rebuild_index in hash-index-storage/bitcask.py

**Date:** 2026-05-28
**Time:** 18:57

# `BitcaskStore._rebuild_index`

## Purpose

This is the startup recovery method for the Bitcask store. When a `BitcaskStore` is opened against an existing data directory, `_rebuild_index` reconstructs the in-memory hash index (`self.keydir`) by replaying all data files from disk. This is necessary because Bitcask's entire read path depends on the in-memory index — without it, the store has no way to locate any key's value on disk.

It exists because Bitcask trades startup cost for runtime simplicity: the index is never persisted as a standalone structure (except via optional hint files), so it must be rebuilt on every open.

## Contract

**Preconditions:**
- `self.keydir` is empty (or at least expected to be overwritten) — the method populates it additively without clearing first.
- `self.data_dir` exists and contains valid `.data` files for each ID in `file_ids`.
- `self.file_handles` is initialized (empty dict is fine — readers are opened lazily by `_get_reader`).

**Postconditions:**
- `self.keydir` contains exactly one `KeyEntry` per live key across all processed files. Tombstoned keys are absent.
- Files are processed in ascending ID order, so later files' entries overwrite earlier ones — matching Bitcask's append-only semantics where the latest write wins.

**Invariant:** After completion, for every key `k` in `self.keydir`, `keydir[k]` points to the most recent non-tombstone record for `k` across all scanned files.

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `file_ids` | `list[int]` | Numeric IDs of data files to scan (e.g., `[0, 1, 3]` corresponding to `0.data`, `1.data`, `3.data`). Gaps are fine. The list does not need to be pre-sorted — the method sorts it internally. |

**Edge cases:**
- Empty list: the loop body never executes; `keydir` stays empty. This is valid but never happens in practice — the caller (`__init__`) only calls this when `existing_ids` is non-empty.
- Single-element list: works correctly, just scans one file.

## Return Value

`None`. The method mutates `self.keydir` in place.

## Algorithm

1. **Sort file IDs** — `sorted(file_ids)` ensures files are processed oldest-first. This is critical: if key `"x"` appears in both file 0 and file 2, file 2's entry overwrites file 0's, leaving the latest value in the index.

2. **For each file, choose the fast or slow path:**
   - **Fast path (hint file exists):** Call `_load_hint_file(fid)`. Hint files contain pre-computed index entries (file_id, offset, size, timestamp, key) without the actual values, so they're smaller and faster to parse — no need to skip over value bytes.
   - **Slow path (no hint file):** Call `_scan_data_file(fid)`. This sequentially reads every record header in the `.data` file, extracts the key and metadata, skips the value bytes, and builds index entries. It also handles tombstones (records with `val_size == 0`) by removing the key from `keydir`.

3. The method itself contains no merging logic — ordering guarantees correctness. Each delegate (`_load_hint_file` or `_scan_data_file`) writes directly into `self.keydir`, and later files naturally overwrite stale entries from earlier files.

## Side Effects

- **Mutates `self.keydir`**: adds/removes entries for every key encountered.
- **Opens file handles**: `_scan_data_file` calls `_get_reader`, which lazily opens `.data` files and caches them in `self.file_handles`. These handles stay open for the lifetime of the store.
- **Reads from disk**: both paths perform file I/O. For large stores without hint files, this can be slow — the entire dataset must be scanned sequentially.

## Error Handling

There is no explicit error handling. The method will propagate:
- `FileNotFoundError` if a `.data` file for a given ID doesn't exist on disk (broken invariant from caller).
- `struct.error` if a data or hint file is corrupted (truncated headers, malformed records).
- `UnicodeDecodeError` if key bytes are not valid UTF-8.

None of these are caught — a corrupt file means a crash at startup. This is a deliberate design choice for a reference implementation: fail loudly rather than silently lose data.

## Usage Patterns

Called exactly once, from `__init__`, and only when existing data files are found:

```python
existing_ids = self._find_file_ids()
if existing_ids:
    self._rebuild_index(existing_ids)
```

The caller then sets `self.active_file_id = max(existing_ids)` — the rebuild must complete before the active file is opened for writes. There's no locking; this assumes single-process access (standard Bitcask constraint).

## Dependencies

| Dependency | Role |
|------------|------|
| `self._hint_path(fid)` | Resolves `<data_dir>/<fid>.hint` |
| `self._load_hint_file(fid)` | Fast-path index loading from precomputed hint files |
| `self._scan_data_file(fid)` | Slow-path sequential scan of raw data files |
| `os.path.exists` | Checks for hint file presence |
| `self.keydir` | Target dict being populated |

## Assumptions Not Enforced by Types

1. **File IDs correspond to actual files on disk.** Nothing prevents passing an ID with no matching `.data` file — it will crash inside `_scan_data_file`.
2. **Hint files are consistent with data files.** If a hint file is stale or corrupt, the index will be wrong, but no validation is performed.
3. **Tombstone handling differs between paths.** `_scan_data_file` removes tombstoned keys from `keydir`; `_load_hint_file` does not — hint files are only written during compaction, which already strips tombstones. If a hint file somehow contained a tombstoned key, it would appear as live.
4. **Single-writer assumption.** No file locking or CRC checks — concurrent writers would corrupt the index silently.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:_scan_data_file` — The slow-path record parser that handles tombstones and builds index entries from raw data
- [function] `hash-index-storage/bitcask.py:compact` — Produces the hint files that `_rebuild_index` uses for fast startup; understanding compaction explains when and why hint files exist
- [function] `hash-index-storage/bitcask.py:_load_hint_file` — The fast-path loader whose binary format differs from data files; worth comparing the two struct layouts
- [general] `bitcask-startup-cost` — How real Bitcask implementations (Riak's original, or modern forks) mitigate the O(n) startup scan through hint files, memory-mapped I/O, and parallel file scanning
- [file] `hash-index-storage/test_bitcask.py` — Tests that exercise the rebuild path, especially after compaction and across file rotations

## Beliefs

- `rebuild-processes-files-in-order` — `_rebuild_index` sorts file IDs ascending so that newer records overwrite older ones in the keydir, matching append-only last-write-wins semantics
- `hint-file-is-fast-path` — When a `.hint` file exists for a data file, `_rebuild_index` skips the full data scan and loads pre-computed index entries, avoiding the cost of reading value bytes
- `no-corruption-detection-on-rebuild` — Neither `_scan_data_file` nor `_load_hint_file` validates CRC checksums or structural integrity; a corrupt file crashes the process rather than recovering gracefully
- `tombstone-handling-asymmetry` — `_scan_data_file` removes tombstoned keys from `keydir` during rebuild, but `_load_hint_file` unconditionally inserts entries, relying on compaction having already stripped tombstones before writing the hint file

