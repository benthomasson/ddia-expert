# Topic: The original Bitcask paper by Riak describes hint files, merge semantics, and the keydir; compare this implementation's choices against the paper's recommendations

**Date:** 2026-05-29
**Time:** 06:55

I have enough from the observations to do a thorough comparison. Let me work through the three pillars of the Bitcask paper against these two implementations.

---

# Bitcask Paper vs. Implementation Choices

The Bitcask paper (Sheehy & Smith, 2010) specifies three core mechanisms: the **keydir** (an in-memory hash table), **hint files** (for fast startup), and **merge** (background compaction of immutable files). This repo contains two independent implementations — `hash-index-storage/bitcask.py` and `log-structured-hash-table/bitcask.py` — that make meaningfully different trade-offs against the paper.

## The Keydir

The paper defines the keydir entry as a fixed-size tuple: `(file_id, value_sz, value_pos, tstamp)`. Every live key maps to exactly one of these. The keydir is the **only** index — there are no secondary structures.

**`hash-index-storage/bitcask.py`** follows this closely. Its `KeyEntry` dataclass (lines 20–24) stores `(file_id, offset, size, timestamp)` — four fields, matching the paper's intent. The one difference: `size` is the total record size (header + key + value), not just `value_sz`. This means a `get()` reads the entire record in one call to `_read_record` (line 98), which is arguably simpler than computing value position from a separate value offset. The keydir is named `self.keydir` (line 35) and is a plain `dict[str, KeyEntry]`.

**`log-structured-hash-table/bitcask.py`** departs significantly. Its index (line 41) stores only `(filepath, offset)` — **no timestamp, no size**. This has two consequences:

1. **Reads require two disk ops**: `get()` (lines ~170–180) must first read the header to learn `key_size` and `value_size`, then read the payload. The paper's design avoids this by keeping `value_sz` in the keydir so you can issue a single positional read of the exact right length.

2. **Merge can't compare timestamps from the keydir alone**. The paper's merge process uses the keydir's timestamp to determine which version of a key is live. Without it, this implementation must rely on the ordering of segment scans (newest segment wins) or re-read records from disk.

**Verdict**: The `hash-index-storage` version is paper-faithful. The `log-structured-hash-table` version trades memory efficiency (smaller index entries) for slower reads and a weaker merge primitive.

## Hint Files

The paper describes hint files as produced **during merge**, one per merged data file. Each hint entry contains `(tstamp, ksz, value_sz, value_pos, key)` — everything needed to reconstruct the keydir entry without reading the data file's values. This is the critical startup optimization: without hint files, recovery requires scanning every byte of every data file.

**`hash-index-storage/bitcask.py`** implements this correctly. Its hint format (line 13) is `HINT_FORMAT = "<IQId"` — `(file_id, offset, size, timestamp)` — plus a `key_size` and `key` appended after each fixed entry (lines 157–165 in `_write_hint_file`). Hint files are written automatically during compaction (lines 261, 280). On startup, `_rebuild_index` (lines 107–114) checks for hint files first and falls back to `_scan_data_file` only when none exist. This matches the paper's design.

One subtle divergence: the hint entries include `file_id` even though the paper's hint files are inherently per-data-file (so the file_id is implicit from the filename). This implementation stores `entry.file_id` in hint entries (line 161), which is redundant for 1:1 hint-to-data mappings but becomes useful if a single hint file could reference multiple data files — which this implementation's merge process can produce when it rotates mid-compaction.

**`log-structured-hash-table/bitcask.py`** has a much thinner hint file. Its format (line 12) is `HINT_ENTRY_FMT = "!II"` — just `(key_size, offset)`. No timestamp, no value size. This mirrors the stripped-down index: since the index only stores `(path, offset)`, the hint file only needs to replay that. But it means hint files can't fully reconstruct a paper-spec keydir.

More importantly, hint file creation is **not automatic during compaction**. The test file (line ~115 of the test) shows an explicit `store.create_hint_files()` call after `compact()`. The paper is clear that hint files should be a side-effect of merge — they come "for free" since the merge process already iterates over every live key. Making it a separate step means there's a window where merged data files exist without hint files, leading to slow restarts.

The `_load_hint_file` (lines 108–117) also doesn't handle tombstones. This works only because hint files are created after compaction (which strips tombstones), but it's fragile — if someone called `create_hint_files()` on an uncompacted segment, deleted keys would reappear in the index on restart.

**Verdict**: `hash-index-storage` gets hint files right (automatic, during merge, with enough metadata). `log-structured-hash-table` treats them as an afterthought.

## Merge Semantics

The paper's merge process: iterate over all immutable (non-active) data files from oldest to newest, keeping only the latest version of each key, discarding tombstones, writing new data files, and producing hint files alongside them. The active file is never touched. Old files are deleted after the merge completes.

**`hash-index-storage/bitcask.py`** `compact()` (line 194) follows this closely:

- Only compacts immutable files: `immutable_ids = [fid for fid in all_ids if fid != self.active_file_id]` (line 198).
- Scans immutable files and checks whether each key's current keydir entry still points to the immutable set (line 231: `entry = self.keydir.get(key)`) — this is how it identifies stale entries.
- Writes merged data to new files with rotation at `max_file_size` (line 259), and writes hint files for each merged file (lines 261, 280).
- Deletes old immutable files after the merge.
- Updates the active file ID to be after the merged files (line 295).

This is essentially the paper's algorithm. One deviation: the paper describes merge as iterating data files and keeping the newest version by timestamp comparison. This implementation instead uses the keydir as the authoritative source of "which version is live" — if the keydir points to this file/offset, the record is live; otherwise it's stale. This is actually **more correct** than pure timestamp comparison, since it handles the case where two records have identical timestamps.

**`log-structured-hash-table/bitcask.py`** adds auto-compaction (lines 160–163 in `put()`): when the number of frozen segments exceeds `auto_compact_threshold`, compaction triggers automatically. The paper mentions this as a possible background process. The full compaction logic isn't visible in the first 200 lines, but the test shows it returns a count of stale entries, and disk usage decreases afterward.

## Other Paper Recommendations

| Paper Feature | `hash-index-storage` | `log-structured-hash-table` |
|---|---|---|
| **CRC checksums** | Missing — no integrity check on records | Present — CRC32 in header (line 10), validated on read with `CorruptionError` (lines 95–100) |
| **Tombstone sentinel** | Empty string `""` (fragile — can't store empty values) | Dedicated `TOMBSTONE = b"__BITCASK_TOMBSTONE__"` (line 7) — better but still occupies the value namespace |
| **fsync control** | `sync_writes` parameter (line 31), fsync after each write (line 93) | No fsync — just `flush()` (line 144). Crash-unsafe |
| **Partial write recovery** | Stops scanning at incomplete record (line 126) | Stops scanning at incomplete record or CRC mismatch. Has explicit test coverage (test line ~147) |
| **Value type** | Strings only | Bytes — more general, closer to paper's opaque-value semantics |

## Summary

Neither implementation is a complete Bitcask. The `hash-index-storage` version gets the architecture right (keydir shape, hint file generation during merge, merge semantics) but skips CRC integrity checking. The `log-structured-hash-table` version adds CRC and auto-compaction but weakens the keydir (missing timestamp and size), treats hint files as optional and manual, and doesn't fsync. If you were building a production system, you'd want the `hash-index-storage` architecture with the `log-structured-hash-table` CRC and corruption handling grafted on.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — The full merge implementation (lines 194–300+) is the most paper-faithful part of the codebase; trace how it decides which records are stale and how it atomically switches from old to new files
- [function] `log-structured-hash-table/bitcask.py:compact` — Compare this implementation's compaction (beyond line 200) against the hash-index-storage version, particularly how it handles the index update and old segment cleanup without timestamps in the index
- [general] `tombstone-semantics` — The empty-string tombstone in `hash-index-storage` means you literally cannot store an empty value; compare with the dedicated sentinel in `log-structured-hash-table` and consider what the paper recommends (a flag in the record header)
- [file] `log-structured-hash-table/test_bitcask.py` — The partial-write recovery and CRC corruption tests (lines ~147–180) exercise crash scenarios that `hash-index-storage` has no coverage for
- [general] `concurrent-merge-safety` — Neither implementation handles concurrent readers during merge; the paper describes an atomic switchover where old files are deleted only after the keydir is updated — trace whether the `compact()` methods maintain this invariant or have a window where reads could fail

## Beliefs

- `hash-index-keydir-has-four-fields` — The `hash-index-storage` KeyEntry stores (file_id, offset, size, timestamp), matching the paper's keydir entry shape, while `log-structured-hash-table` stores only (filepath, offset)
- `hint-files-automatic-only-in-hash-index` — Only `hash-index-storage/bitcask.py` generates hint files automatically during compaction; `log-structured-hash-table` requires an explicit `create_hint_files()` call
- `no-crc-in-hash-index-storage` — The `hash-index-storage` record format (`<dII` + key + value) contains no checksum; data corruption is undetectable on read
- `empty-string-is-tombstone` — In `hash-index-storage/bitcask.py`, deletes write an empty-string value and `get()` returns `None` for empty values (line 181), making it impossible to store a legitimate empty-string value
- `compact-skips-active-file` — Both implementations exclude the active/current file from compaction, matching the paper's requirement that only immutable files are merged

