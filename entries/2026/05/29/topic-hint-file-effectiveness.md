# Topic: Measure how much hint files actually reduce recovery time by comparing `_load_hint_file` vs `_scan_data_file` on realistic data sizes

**Date:** 2026-05-29
**Time:** 12:15

# Hint Files vs Data File Scanning in Bitcask Recovery

## The Core Trade-off

When a Bitcask store restarts, it must rebuild its in-memory hash index (`keydir`) from disk. There are two paths, chosen per-file in `_rebuild_index` (`hash-index-storage/bitcask.py:108-114`):

```python
def _rebuild_index(self, file_ids: list[int]):
    for fid in sorted(file_ids):
        hint_path = self._hint_path(fid)
        if os.path.exists(hint_path):
            self._load_hint_file(fid)
        else:
            self._scan_data_file(fid)
```

The hint file is preferred when it exists; the data file scan is the fallback.

## Why Scanning Data Files Is Expensive

`_scan_data_file` (`hash-index-storage/bitcask.py:116-133`) must read **every byte** of the data file sequentially:

1. Read the 16-byte header (`HEADER_FORMAT = "<dII"` — timestamp, key_size, value_size)
2. Read the full key bytes
3. **Read and discard the value bytes** (`reader.read(val_size)` at line 128 — the value is skipped but still must be read to advance the file position)
4. Check for tombstones (empty values) and update/remove from `keydir`

The critical cost: **values are read from disk even though they're thrown away**. If you have 100,000 keys each with 1 KB values, the scan reads ~100 MB of value data that contributes nothing to the index. The I/O is proportional to the **total data file size**, not the number of keys.

The `log-structured-hash-table/bitcask.py` variant (`_scan_segment`, lines 89-107) has the same structure but adds CRC verification via `zlib.crc32` on every record, making it even more expensive per record.

## Why Hint Files Are Fast

`_load_hint_file` (`hash-index-storage/bitcask.py:135-147`) reads a compact file that contains **only index metadata**:

```
HINT_FORMAT = "<IQId"  # file_id(uint32), offset(uint64), size(uint32), timestamp(double)
```

Each hint entry is: 20 bytes of fixed header (`HINT_HEADER_SIZE`) + 4 bytes for key length + the key bytes. **No values are stored in the hint file at all.**

For the same 100,000 keys with average 10-byte key names, the hint file is roughly:
- `100,000 × (20 + 4 + 10)` = ~3.2 MB

Compare that to the ~100 MB data file. The hint file is **~30x smaller** for this workload, and the ratio grows linearly with value size.

## Structural Differences

| Aspect | `_scan_data_file` | `_load_hint_file` |
|--------|-------------------|-------------------|
| Reads values | Yes (discards them) | No values stored |
| Tombstone handling | Detects `val_size == 0`, removes from keydir | No tombstone awareness — only live keys written |
| I/O proportional to | Total data file size | Number of live keys × key size |
| Per-record overhead | Parse header + seek past value | Parse fixed-size hint entry |
| File handle | Uses shared `_get_reader` | Opens/closes its own handle |
| CRC checks | None in hash-index variant; yes in log-structured variant | None |

## The `_load_hint_file` quirk in the log-structured variant

The `log-structured-hash-table/bitcask.py` version (lines 110-121) has a simpler hint format — just `key_size` and `offset` (`HINT_ENTRY_FMT = "!II"`) — with no timestamp or record size. This means it stores even less per entry but also provides less metadata for the index. It maps `key → (seg_path, offset)` rather than the richer `KeyEntry(file_id, offset, size, timestamp)` used in the hash-index variant.

## What's Missing for Actual Measurement

The observations are sufficient to explain the *structural* advantage, but to **measure** the actual recovery time reduction you'd need:

1. **A benchmark harness** — neither implementation includes timing instrumentation around `_rebuild_index`, `_load_hint_file`, or `_scan_data_file`
2. **Realistic data generation** — a script to populate stores with known key/value size distributions (e.g., 10-byte keys, 1 KB values, 100K–1M records)
3. **Hint file creation trigger** — `_write_hint_file` (`hash-index-storage/bitcask.py:149-157`) is only called during compaction, so you'd need to compact before benchmarking hint-based recovery
4. **The rest of `bitcask.py`** — lines 200-321 of `hash-index-storage/bitcask.py` were not provided; the `compact()` method (which triggers hint file creation) is cut off at line 200

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — This is where `_write_hint_file` gets called; understanding compaction is essential to knowing when hint files exist
- [function] `hash-index-storage/bitcask.py:_write_hint_file` — The hint file serialization format determines what metadata survives vs what's lost compared to a full data scan
- [file] `hash-index-storage/test_bitcask.py` — Contains `test_hint_files` (line 61) and a rebuild-without-hints test (line 93) that verify both recovery paths
- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — Compare CRC-verified scanning in the log-structured variant to understand the additional CPU cost hint files avoid
- [general] `hint-file-tombstone-handling` — Hint files in this implementation don't encode tombstones, meaning they're only written for live keys post-compaction — explore what happens if hint files are written for uncompacted files

## Beliefs

- `hint-file-omits-values` — Hint files store only key + location metadata (file_id, offset, size, timestamp), never values, making them proportionally smaller as value sizes grow
- `scan-reads-then-discards-values` — `_scan_data_file` must `reader.read(val_size)` to advance the file cursor past each value, paying I/O cost for data it never uses
- `hint-files-only-after-compaction` — `_write_hint_file` is called from `compact()`, so hint files only exist for compacted (immutable) data files, never for the active file
- `rebuild-prefers-hint-over-scan` — `_rebuild_index` checks for a `.hint` file first and only falls back to `_scan_data_file` when no hint file exists for a given file_id
- `two-bitcask-variants-differ-in-hint-format` — The hash-index variant stores `(file_id, offset, size, timestamp)` per hint entry while the log-structured variant stores only `(key_size, offset)`, trading metadata richness for compactness

