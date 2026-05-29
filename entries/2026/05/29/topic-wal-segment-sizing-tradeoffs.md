# Topic: How `max_file_size` affects recovery time (more segments = faster truncation but more files to scan on replay) and write amplification

**Date:** 2026-05-29
**Time:** 08:27

I have enough from the observations to give a thorough explanation. Let me work with what's already gathered.

---

# How `max_file_size` Affects Recovery Time and Write Amplification

## The Core Tradeoff

`max_file_size` controls when a log or data file is rotated into a new segment. It appears in three implementations across this codebase, and in every case it creates the same fundamental tension: **smaller segments speed up truncation but slow down recovery; larger segments do the opposite.**

## The Mechanism: Rotation

All three implementations follow the same pattern — when the active file reaches `max_file_size`, a new file is created:

**WAL** (`write-ahead-log/wal.py:138`):
```python
def _maybe_rotate(self):
    if self._fd and self._fd.tell() >= self._max_file_size:
        self._rotate()
```

**Bitcask (hash-index)** (`hash-index-storage/bitcask.py:101-103`):
```python
def _maybe_rotate(self):
    if self.active_file.tell() >= self.max_file_size:
        ...
```

**Bitcask (log-structured)** (`log-structured-hash-table/bitcask.py:151-153`):
```python
if self._active_file.tell() >= self._max_segment_size:
    self._rotate_segment()
```

The default in both WAL and hash-index Bitcask is **10 MB** (`wal.py:65`, `bitcask.py:29`). The log-structured variant defaults to **1 MB** (`log-structured-hash-table/bitcask.py:37`). Tests use tiny values (100–4096 bytes) to force rotation.

## Effect 1: Recovery Time (the two-sided coin)

### Why smaller segments help truncation

WAL truncation (`wal.py:178+`) works by scanning each file, keeping only records with `seq_num > up_to_seq`, and deleting files that have no surviving records. When segments are small:

- Most old segments contain **only** records below the truncation point, so they can be deleted as whole files — no rewriting needed.
- The "partial" segment (straddling the truncation boundary) is smaller, so rewriting it is cheaper.

### Why smaller segments hurt replay

Recovery requires scanning **every** segment file to rebuild state. Look at the three recovery paths:

1. **WAL** `_recover_seq_num` (`wal.py:85-96`) iterates `self._wal_files()` — every `.wal` file — and reads every record to find the max sequence number. More files = more `open()` calls, more seeks, more syscall overhead.

2. **Bitcask (hash-index)** `_rebuild_index` (`bitcask.py:109-114`) loops over all file IDs and calls `_scan_data_file` for each one. Each scan opens the file, reads it front-to-back, and populates `keydir`. More files = more iterations.

3. **Bitcask (log-structured)** `_recover` (`log-structured-hash-table/bitcask.py:64-83`) does the same — iterates every segment, calling `_scan_segment` or `_load_hint_file` per segment.

The per-file overhead matters: each segment requires an `open()` syscall, a file handle, and sequential I/O from position 0. With a 10 MB `max_file_size` and 1 GB of data, you scan ~100 files. Drop to 100 KB and you're scanning ~10,000 files. The total bytes read is the same, but the syscall and seek overhead scales linearly with file count.

### Hint files partially mitigate this

Both Bitcask variants support **hint files** that store the index compactly without the values. The hash-index version checks for hints before falling back to a full scan (`bitcask.py:111-114`):

```python
if os.path.exists(hint_path):
    self._load_hint_file(fid)
else:
    self._scan_data_file(fid)
```

Hint files reduce the bytes-read cost but not the files-to-open cost — you still have one hint file per data file.

## Effect 2: Write Amplification

Write amplification measures how many times data is physically written compared to logical writes. `max_file_size` affects this through **compaction frequency**.

### More segments trigger more compaction

The log-structured Bitcask has an explicit `auto_compact_threshold` (`log-structured-hash-table/bitcask.py:38`). After every `put`, it checks whether the number of frozen (non-active) segments exceeds the threshold (`bitcask.py:160-162`):

```python
frozen = self._frozen_segment_paths()
if len(frozen) >= self._auto_compact_threshold:
    self.compact()
```

Smaller `max_file_size` → segments fill faster → the threshold is hit more often → compaction runs more frequently. Each compaction rewrites all live data in the frozen segments. If data is being overwritten at a moderate rate, smaller segments mean the same records get rewritten during compaction more often before they'd naturally be superseded — that's **higher write amplification**.

### Compaction itself respects the size limit

In `hash-index-storage/bitcask.py:259`, even the merged output file rotates when it hits `max_file_size`:

```python
if merged_file.tell() >= self.max_file_size:
```

This means compaction can't consolidate everything into one big file. Smaller limits produce more post-compaction files, which feeds back into the recovery cost.

## The Design Space

| `max_file_size` | Truncation | Recovery | Write Amplification | File Count |
|---|---|---|---|---|
| **Small** (e.g., 100 KB) | Fast — most files deleted wholesale | Slow — many files to open and scan | Higher — compaction triggers more often | Many |
| **Large** (e.g., 100 MB) | Slow — partial-file rewrite is expensive | Fast — few files to scan | Lower — fewer compaction cycles | Few |

The 10 MB default in both WAL and hash-index Bitcask is a practical middle ground for most workloads. The 1 MB default in the log-structured variant reflects that it relies more heavily on compaction (with hint files to offset the recovery cost).

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:replay` — How WAL replay filters records by `after_seq` and how checkpoint records interact with recovery
- [function] `hash-index-storage/bitcask.py:compact` — The full compaction loop including how it handles tombstones and produces hint files, directly affecting write amplification
- [function] `log-structured-hash-table/bitcask.py:compact` — Compare the log-structured variant's compaction with the hash-index version — different merge strategies, same segment-size pressures
- [file] `log-structured-merge-tree/lsm.py` — How LSM trees take the segment-size tradeoff further with tiered compaction and a `compaction_threshold` parameter (line 204)
- [general] `hint-file-effectiveness` — Measure how much hint files actually reduce recovery time by comparing `_load_hint_file` vs `_scan_data_file` on realistic data sizes

---

## Beliefs

- `wal-default-segment-10mb` — WAL and hash-index Bitcask both default `max_file_size` to 10 MB (`wal.py:65`, `bitcask.py:29`), while the log-structured Bitcask defaults to 1 MB (`log-structured-hash-table/bitcask.py:37`)
- `recovery-scans-all-segments` — All three implementations scan every segment file on startup to rebuild state (`wal.py:85-96`, `bitcask.py:109-114`, `log-structured-hash-table/bitcask.py:64-83`), making recovery time proportional to segment count
- `compaction-respects-size-limit` — Bitcask compaction output is itself subject to `max_file_size` rotation (`hash-index-storage/bitcask.py:259`), so smaller limits produce more post-compaction files
- `auto-compact-triggers-on-frozen-count` — The log-structured Bitcask triggers compaction when frozen segment count exceeds `auto_compact_threshold` (`log-structured-hash-table/bitcask.py:160-162`), directly coupling segment size to compaction frequency
- `hint-files-skip-value-scanning` — Both Bitcask variants use hint files to bypass full data-file scans during index rebuild, but the number of hint files still scales with segment count

