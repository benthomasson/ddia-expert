# Topic: The `log-structured-hash-table` hint format has no way to represent tombstones (it's just key_size + offset), so deleted keys from compaction may also have subtle semantics worth tracing

**Date:** 2026-05-29
**Time:** 08:53

# The Hint File Tombstone Gap in `log-structured-hash-table`

## The Core Problem

The hint file format at `log-structured-hash-table/bitcask.py:13-14` stores only two fields per entry:

```python
HINT_ENTRY_FMT = "!II"  # key_size, offset
```

Meanwhile, tombstones are represented as a magic value in the data file (`bitcask.py:9`):

```python
TOMBSTONE = b"__BITCASK_TOMBSTONE__"
```

The hint format has no field to distinguish "this offset points to a live record" from "this offset points to a tombstone." This creates a divergence between two recovery paths that should produce identical results.

## The Two Recovery Paths

**Path A — Scanning the data file** (`_scan_segment`, lines 96–114) correctly handles tombstones:

```python
if value == TOMBSTONE:
    self._index.pop(key, None)   # deleted key is removed from the index
else:
    self._index[key] = (seg_path, offset)
```

**Path B — Loading a hint file** (`_load_hint_file`, lines 116–127) blindly trusts every entry:

```python
key = key_bytes.decode("utf-8")
self._index[key] = (seg_path, offset)   # always adds — no tombstone check possible
```

These two paths are selected during recovery (`_recover`, lines 82–96): if a `.hint` file exists for a segment, it's used; otherwise the segment is scanned. The intent is that both paths produce the same index. For tombstones, they don't.

## How Deleted Keys Resurrect

Consider this sequence:

1. Segment 1 contains `put("user", b"Alice")`
2. Segment 2 contains `delete("user")` — writes `TOMBSTONE` value
3. Hint files are generated for both segments

On recovery:
- Segment 1's hint file adds `user → (seg1, offset)` to the index
- Segment 2's hint file **also adds** `user → (seg2, tombstone_offset)` to the index, overwriting segment 1's entry

Now `get("user")` reads the record at `tombstone_offset`, finds the payload is `b"__BITCASK_TOMBSTONE__"`, and **returns it as the value**. The `get` method (lines 164–179) performs no tombstone check — it returns `payload[key_size:]` unconditionally. A deleted key has come back to life, and worse, its value is the raw tombstone sentinel bytes.

Had `_scan_segment` processed segment 2 instead, it would have called `self._index.pop("user", None)`, correctly removing the key.

## Why Compaction Masks This (Usually)

This bug is likely latent rather than actively triggered, because compaction should strip tombstones from the output segment — dead keys simply aren't written to the merged file. If `create_hint_files()` is only called on compacted segments (as the test at lines 104–113 suggests), the hint files would never reference tombstone records.

But the code offers no structural guarantee of this. `create_hint_files()` and `compact()` are independent public methods. Calling `create_hint_files()` on an uncompacted segment that contains deletions would produce a hint file with phantom entries.

## Contrast with `hash-index-storage`

The sibling implementation at `hash-index-storage/bitcask.py` has the same structural gap — its `_load_hint_file` (lines ~148–160) also unconditionally adds entries. But it has a defense-in-depth layer: its `get()` method (lines ~175–182) checks `if value == "": return None`, so even if a tombstone leaks into the index through a hint file, reads return `None` rather than sentinel bytes. The `log-structured-hash-table` implementation lacks this second guard.

## What's Missing from the Observations

The `compact()` and `create_hint_files()` methods (likely beyond line 200 of `bitcask.py`) weren't included. Whether compaction writes tombstones to the output segment — and whether `create_hint_files` iterates the index or scans the segment — determines whether this bug is practically reachable or only theoretical.

---

## Topics to Explore

- [function] `log-structured-hash-table/bitcask.py:compact` — Does compaction strip tombstones from the output, or could they leak into compacted segments?
- [function] `log-structured-hash-table/bitcask.py:create_hint_files` — Does this iterate the in-memory index (safe) or scan segment files (unsafe for tombstones)?
- [function] `hash-index-storage/bitcask.py:compact` — Compare how this implementation handles tombstones during merge to see if the defense-in-depth pattern extends to compaction
- [general] `hint-file-tombstone-encoding` — How does real Bitcask (Riak's) hint format encode tombstones? The original paper's hint format includes a value size field that can signal deletion with size=0
- [file] `log-structured-hash-table/test_bitcask.py` — The hint file test (line 104) only tests post-compaction hint creation; a test that creates hints on uncompacted segments with deletions would expose this bug

## Beliefs

- `lsht-hint-no-tombstone-flag` — The `log-structured-hash-table` hint entry format (`!II` = key_size + offset) has no field to distinguish live records from tombstones, making `_load_hint_file` unable to replicate `_scan_segment`'s deletion logic
- `lsht-get-no-tombstone-guard` — `BitcaskStore.get()` in `log-structured-hash-table/bitcask.py` returns raw payload bytes without checking for the `TOMBSTONE` sentinel, so an index entry pointing to a tombstone record surfaces `b"__BITCASK_TOMBSTONE__"` as a value
- `lsht-recovery-path-divergence` — `_recover` selects between `_load_hint_file` (no tombstone handling) and `_scan_segment` (with tombstone handling) based on hint file existence, meaning the same segment produces different index state depending on which path runs
- `hash-index-get-tombstone-guard` — `hash-index-storage/bitcask.py`'s `get()` checks `if value == "": return None`, providing a second line of defense that `log-structured-hash-table` lacks

