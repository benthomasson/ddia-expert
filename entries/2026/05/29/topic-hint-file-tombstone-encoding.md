# Topic: How does real Bitcask (Riak's) hint format encode tombstones? The original paper's hint format includes a value size field that can signal deletion with size=0

**Date:** 2026-05-29
**Time:** 12:53

# How Bitcask Hint Files Encode Tombstones

## The Real Bitcask Design

In the original Bitcask paper and Riak's production implementation, the hint file format includes a `value_size` field for each entry. This field serves double duty: it tells you how large the value is on disk (so you can read it without scanning), and it signals tombstones when `value_size == 0`. The hint file is a fast-path for rebuilding the in-memory keydir at startup — instead of scanning every data file record by record, you read the compact hint file that contains just the metadata needed to populate the index.

The critical design choice: because the hint file carries `value_size`, it can distinguish "this key has a live value" from "this key was deleted" **without touching the data file at all**. During index rebuild, a hint entry with `value_size == 0` means "remove this key from the keydir" — the same as encountering a tombstone record in a data file scan.

## What This Codebase Does

### `hash-index-storage/bitcask.py` — Partial Alignment

This implementation gets the data file tombstone encoding right but **misses it in the hint file**.

**Data file side** (lines 135–137): tombstones are correctly detected via `val_size == 0` during `_scan_data_file`:

```python
if val_size == 0:
    # Tombstone - remove from keydir
    self.keydir.pop(key, None)
```

**Hint file side** (lines 13, 143–157): the hint format is `<IQId` — that's `file_id(uint32), offset(uint64), size(uint32), timestamp(double)`. The `size` field here is the **total record size**, not the value size. When `_load_hint_file` reads entries, it unconditionally adds them to the keydir:

```python
self.keydir[key] = KeyEntry(fid, offset, size, ts)
```

There is **no tombstone check** in `_load_hint_file`. If a compacted data file contained tombstone records and a hint file was written for it, those tombstones would be loaded as live entries, pointing to records with empty values. The `get` method (line 179) catches this at read time — `if value == ""` returns `None` — but the keydir is polluted with dead entries that consume memory and produce unnecessary disk reads.

This is a divergence from real Bitcask. In the original design, the hint file either encodes `value_size` so tombstones can be filtered during rebuild, or tombstones are simply omitted from hint files during compaction (since compaction is supposed to drop them).

### `log-structured-hash-table/bitcask.py` — Different Problem

This implementation uses a sentinel value (`TOMBSTONE = b"__BITCASK_TOMBSTONE__"`, line 9) instead of `value_size == 0`. Its hint format (line 12) is `!II` — just `key_size` and `offset`. No value size at all.

The `_load_hint_file` method (lines 115–124) also unconditionally adds entries to the index. Since the hint format doesn't carry a value size or any tombstone flag, it **cannot** distinguish live entries from tombstones without reading the data file — which defeats the purpose of hint files as a fast startup path.

The `_scan_segment` method (lines 89–104) correctly handles tombstones by checking `if value == TOMBSTONE`, but this logic has no equivalent in the hint path.

## The Gap

Neither implementation faithfully reproduces the real Bitcask hint file tombstone encoding. The correct approach would be one of:

1. **Include `value_size` in the hint format** and check for `value_size == 0` during hint file loading (matching the original paper)
2. **Exclude tombstone entries from hint files entirely**, which is safe if hint files are only written during compaction (where tombstones are already being dropped)

The `hash-index-storage` version is closer — it has the right field in its data format and checks it during data file scans — but doesn't carry the signal through to hint files. The `log-structured-hash-table` version uses a sentinel value pattern that's fundamentally incompatible with the hint file optimization.

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:_write_hint_file` — See how hint entries are constructed during compaction and whether tombstones are filtered before writing
- [function] `hash-index-storage/bitcask.py:compact` — The compaction path determines which entries survive into hint files; check whether tombstones are excluded here
- [function] `log-structured-hash-table/bitcask.py:compact` — Compare the two compaction strategies and how each handles tombstone elimination
- [general] `bitcask-paper-hint-format` — Read the original Bitcask paper (Section 3.2) to see the exact hint entry layout: `tstamp, ksz, value_sz, value_pos, key`
- [file] `log-structured-merge-tree/lsm.py` — Contrast the LSM-tree's tombstone handling (which must preserve tombstones across compaction levels) with Bitcask's simpler single-level merge

## Beliefs

- `hash-index-hint-file-ignores-tombstones` — `_load_hint_file` in `hash-index-storage/bitcask.py` unconditionally adds every hint entry to the keydir with no tombstone check, unlike real Bitcask which uses `value_size == 0` in hint entries to signal deletions
- `hash-index-data-scan-detects-tombstones` — `_scan_data_file` at line 135 correctly identifies tombstones via `val_size == 0` and removes the key from the keydir, matching the original Bitcask data file semantics
- `log-structured-hint-format-lacks-value-size` — The log-structured-hash-table hint format (`!II` = key_size + offset) contains no value size or tombstone flag, making it impossible to filter deleted keys during hint-based index rebuild
- `two-tombstone-strategies-diverge` — `hash-index-storage/bitcask.py` uses `value_size == 0` (matching the Bitcask paper) while `log-structured-hash-table/bitcask.py` uses a sentinel byte string `b"__BITCASK_TOMBSTONE__"`, and the two approaches have different implications for hint file correctness

