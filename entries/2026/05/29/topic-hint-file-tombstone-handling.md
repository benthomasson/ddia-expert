# Topic: Hint files in `hash-index-storage` (line 150–160) don't encode tombstone status, so a deleted key with a hint entry would resurrect on restart

**Date:** 2026-05-29
**Time:** 13:01

## Hint Files Don't Encode Tombstone Status

This is a classic Bitcask implementation bug where the hint file format silently drops deletion information, allowing dead keys to come back to life after a restart.

### How Deletions Work

The `get` method at line 175 reveals the tombstone convention — deleted keys are stored as records with an empty-string value:

```python
if value == "":
```

A `delete(key)` presumably appends a record with `value=""` to the data file. The `get` method reads the actual record from disk and checks for this sentinel, returning `None` for tombstones. This works correctly during normal operation.

### The Hint File Shortcut

Hint files exist to speed up startup. Instead of scanning every data file record to rebuild the in-memory `keydir`, the store reads compact hint entries that contain only the metadata needed to locate each key's record.

The format at line 11:

```python
HINT_FORMAT = "<IQId"  # file_id(uint32), offset(uint64), size(uint32), timestamp(double)
```

This encodes where to find a record (`file_id`, `offset`, `size`) and when it was written (`timestamp`). **There is no field indicating whether the record is a tombstone.**

### The Resurrection Bug

`_load_hint_file` (lines 140–155) unconditionally populates the keydir for every entry it reads:

```python
self.keydir[key] = KeyEntry(fid, offset, size, ts)
```

It never reads the actual data record to check whether the value is empty. It can't — that's the whole point of hint files (avoid reading data files on startup).

Here's the dangerous sequence:

1. Key `"user:42"` is written with value `"alice"` → stored in data file 1, hint file 1 records the entry
2. Key `"user:42"` is deleted → a tombstone record (`value=""`) is appended to data file 2
3. **Compaction** merges old files. The compactor correctly drops the tombstone (it's dead data). It writes a new data file and a new hint file — but since the tombstone was dropped, `"user:42"` doesn't appear in the new hint file
4. **Restart.** If any surviving hint file still references the pre-deletion record for `"user:42"`, `_load_hint_file` blindly adds it to `keydir`. There's no newer tombstone entry to overwrite it — that was dropped during compaction
5. `get("user:42")` finds the keydir entry, reads the old data record, and returns `"alice"`. **The key has been resurrected.**

### The Fix

The standard Bitcask approach is to encode tombstone status in the hint format. You'd add a flags field:

```python
HINT_FORMAT = "<IQIdB"  # ...existing fields..., flags(uint8)
```

Then `_load_hint_file` would skip entries (or remove them from `keydir`) when the tombstone flag is set. Alternatively, `_write_hint_file` could simply exclude tombstone entries during compaction — but only if the compaction process guarantees that all older files referencing that key are also removed.

### Why `get()` Doesn't Save You

You might think the `if value == "":` check in `get` provides a safety net. It does — but only if the tombstone record still exists on disk. After compaction drops the tombstone, the hint entry may point to the **original, pre-deletion record** with a real value, or to an invalid offset entirely. The `get` check only works when reading a record that is itself the tombstone.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:_maybe_rotate` — File rotation logic that creates new data files; understanding when hint files get written relative to rotation is key to tracing the bug
- [function] `hash-index-storage/bitcask.py:_read_record` — The record deserialization path that actually checks tombstone values; understanding the on-disk record format completes the picture
- [general] `compaction-and-merge` — The compaction/merge process that drops tombstone records and writes new hint files; this is where the bug becomes exploitable
- [file] `log-structured-merge-tree/lsm.py` — Compare how the LSM-tree implementation handles tombstones during compaction; it likely has similar concerns at the merge boundary
- [function] `hash-index-storage/bitcask.py:put` — Trace the full write path from `put` through `_write_record` to understand how records are laid out and why hint files are a lossy summary

---

## Beliefs

- `hint-format-lacks-tombstone-flag` — The `HINT_FORMAT` struct (`<IQId`) encodes file_id, offset, size, and timestamp but has no field for tombstone/deletion status
- `load-hint-unconditional-keydir-insert` — `_load_hint_file` inserts every hint entry into `keydir` without reading the underlying data record or checking for tombstones
- `tombstone-sentinel-is-empty-string` — Deleted keys are represented as records with an empty-string value, checked only in `get()` when reading from disk
- `hint-file-compaction-resurrection` — If compaction drops tombstone records and no hint entry preserves the deletion, a previously deleted key can reappear in `keydir` after restart

