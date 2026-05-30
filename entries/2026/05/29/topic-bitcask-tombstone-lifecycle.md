# Topic: How tombstones flow through write, compaction, and hint generation in both implementations; the `hash-index-storage` variant filters them during compaction (line 221) while `log-structured-hash-table` uses a sentinel value

**Date:** 2026-05-29
**Time:** 12:38

# Tombstone Flow: `hash-index-storage` vs `log-structured-hash-table`

Both modules implement Bitcask-style append-only stores from DDIA Chapter 3, but they encode and handle tombstones differently. Here's how a delete propagates through each system.

## The Two Tombstone Encodings

| | `hash-index-storage/bitcask.py` | `log-structured-hash-table/bitcask.py` |
|---|---|---|
| **Representation** | Empty value string `""` → `val_size == 0` on disk | Sentinel bytes `b"__BITCASK_TOMBSTONE__"` stored as the value |
| **Detection** | Check `val_size == 0` in the binary header | Compare the value bytes to `TOMBSTONE` constant |
| **Has CRC?** | No — trusts the filesystem | Yes — CRC32 covers `key_bytes + value_bytes` |

The `hash-index-storage` approach is more compact (tombstones are zero-length values), while the `log-structured-hash-table` approach is more explicit (tombstones are self-identifying even when read in isolation).

## Write Path

Both implementations follow the same logical flow: `delete(key)` calls `_write_record(key, tombstone_value)` and then removes the key from the in-memory index.

**`hash-index-storage`**: `delete()` calls `_write_record(key, "")` — an empty string value. The method packs a header with `val_size = 0`, appends `header + key_bytes` (no value bytes), then the caller pops the key from `keydir`. See the entry on `_write_record` — it notes: *"Deletions are encoded as records with an empty value string (`""`)"*.

**`log-structured-hash-table`**: `delete()` calls `_write_record(key, TOMBSTONE)` where `TOMBSTONE = b"__BITCASK_TOMBSTONE__"`. The CRC is computed over `key_bytes + TOMBSTONE` bytes. The caller then removes the key from `_index`. The `put` entry explicitly warns: *"`put` does not reject `TOMBSTONE` as a value; passing it creates a record indistinguishable from a delete"* — a correctness hazard unique to the sentinel approach.

In both cases, the key is immediately invisible to `get()` because it's removed from the in-memory index. The on-disk tombstone only matters during recovery and compaction.

## Compaction

This is where the implementations diverge most clearly.

### `hash-index-storage` — Structural filtering (line ~221)

Compaction in `hash-index-storage/bitcask.py` scans immutable files in ascending ID order and builds a `latest` dict. When it encounters a record with `val_size == 0`, it removes that key from `latest`:

```python
if val_size == 0:
    # Tombstone - remove during compaction
    latest.pop(key, None)
```

This is **structural detection** — the tombstone is identified purely by examining the binary header field, not by reading the value. The key is filtered out of `latest`, so it never gets written to the merged output file. Tombstones are eliminated as a natural consequence of the compaction scan.

A second filter (Phase 3) then cross-references `latest` against the `keydir` to exclude keys whose latest value lives in the active file. The combined effect: only keys that are both (a) not tombstoned and (b) still pointed to by `keydir` within immutable files survive into the compacted output.

### `log-structured-hash-table` — Sentinel filtering

Compaction in `log-structured-hash-table/bitcask.py` follows the same scan-then-filter structure, but detects tombstones by comparing the value to the `TOMBSTONE` sentinel. Tombstones remove keys from the `live_entries` dict just as in the other implementation — the difference is *how* they're recognized.

The compaction entry states: *"Tombstones remove keys from `live_entries`. The oldest-to-newest ordering matters: later writes for the same key overwrite earlier ones."*

Then, just like `hash-index-storage`, a second filter checks each entry against `self._index` to verify the frozen-segment record is still the canonical version.

## Hint File Generation

Hint files accelerate startup by storing only index metadata (key + offset), skipping values entirely. But the two implementations generate them at different times and filter tombstones differently.

### `hash-index-storage` — Generated *during* compaction

`_write_hint_file` is called directly inside `compact()`, at two points: when a merged file is rotated (reaches `max_file_size`), and at the end of compaction. Since compaction has already filtered out tombstones (they were popped from `latest`), the hint entries only contain live records. Tombstones **cannot appear** in hint files because they were eliminated before the write loop began.

The hint format includes `file_id`, `offset`, `size`, and `timestamp` per entry (24-byte header + key bytes). No values.

### `log-structured-hash-table` — Generated *separately* via `create_hint_files()`

This implementation decouples hint generation from compaction. `compact()` does **not** call `create_hint_files()`; the caller must invoke it explicitly (e.g., after compaction or before shutdown).

`create_hint_files()` scans each frozen segment record by record and applies two filters:

1. **Skip tombstones** — records whose value equals `TOMBSTONE` are excluded.
2. **Index check** — only emit a hint entry if `self._index[key]` still points to this exact segment and offset.

The entry states the invariant clearly: *"Hint files never contain tombstones or stale entries."*

The hint format is simpler here: just `key_size` + `offset` (8 bytes) followed by key bytes. No timestamp, no file ID (the hint file is per-segment, so the segment identity is implicit).

## Summary of the Flow

```
DELETE(key)
  ├── hash-index-storage: _write_record(key, "")     → val_size=0 on disk
  └── log-structured-hash-table: _write_record(key, TOMBSTONE) → 20-byte sentinel on disk
  
  Both: remove key from in-memory index → invisible to get()

COMPACTION
  ├── hash-index-storage: val_size == 0? → pop from latest dict → never written to merged file
  └── log-structured-hash-table: value == TOMBSTONE? → pop from live_entries → never written

HINT FILES
  ├── hash-index-storage: written inside compact() — tombstones already gone, no filtering needed
  └── log-structured-hash-table: written by create_hint_files() — explicitly skips TOMBSTONE records
```

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:_scan_data_file` — How tombstones are handled during startup index rebuild (the recovery path mirrors compaction's `val_size == 0` check)
- [function] `log-structured-hash-table/bitcask.py:_recover` — The startup replay that rebuilds the index from segments and hint files; tombstones must be handled here too or deleted keys reappear
- [function] `log-structured-hash-table/bitcask.py:_load_hint_file` — The read-side counterpart to `create_hint_files()`; since hint files exclude tombstones, this method never encounters them
- [general] `tombstone-sentinel-hazard` — The `log-structured-hash-table` variant has no guard preventing a user from `put(key, TOMBSTONE)`, which would silently create a phantom delete — worth understanding the blast radius
- [function] `hash-index-storage/bitcask.py:_load_hint_file` — Compare the hint file format (24-byte header with timestamp) against the simpler `log-structured-hash-table` format (8-byte header without timestamp) and the tradeoffs each makes

## Beliefs

- `hash-index-tombstone-is-zero-length-value` — In `hash-index-storage/bitcask.py`, tombstones are records with `val_size == 0`; detection is structural (header inspection), not value comparison.
- `log-structured-tombstone-is-sentinel-bytes` — In `log-structured-hash-table/bitcask.py`, tombstones are records whose value equals `b"__BITCASK_TOMBSTONE__"`; detection requires reading and comparing the full value.
- `hash-index-hint-files-inline-with-compaction` — `hash-index-storage` generates hint files inside `compact()`, so tombstones are already eliminated before hint entries are accumulated.
- `log-structured-hint-files-decoupled-from-compaction` — `log-structured-hash-table`'s `compact()` does not call `create_hint_files()`; hint generation is a separate explicit step that must filter tombstones independently.
- `sentinel-tombstone-lacks-put-guard` — `log-structured-hash-table`'s `put()` does not reject `TOMBSTONE` as a value, meaning a caller can accidentally create a record indistinguishable from a delete that will be silently dropped during compaction and hint generation.

