# Topic: The hint file is written without `fsync` and has no checksums — explore what happens on crash during compaction and whether a partial hint file could silently lose keys on restart

**Date:** 2026-05-29
**Time:** 06:53

# Hint File Crash Safety: Silent Key Loss on Restart

## The Core Problem

Both Bitcask implementations write hint files during compaction as a startup optimization — they let recovery skip scanning data files and instead load the index directly from these lightweight hint files. But the hint files have **no checksums** and are written **without `fsync`**, creating a window where a crash produces a partial hint file that silently omits keys.

## How Hint Files Are Written

In `hash-index-storage/bitcask.py:155-162`, `_write_hint_file` opens the file and writes entries in a simple loop:

```python
def _write_hint_file(self, file_id, entries):
    with open(self._hint_path(file_id), "wb") as f:
        for key, entry in entries:
            key_bytes = key.encode("utf-8")
            f.write(struct.pack(HINT_FORMAT, entry.file_id, entry.offset,
                                entry.size, entry.timestamp))
            f.write(struct.pack("<I", len(key_bytes)))
            f.write(key_bytes)
```

No `fsync` call, no checksum over the file. Compare this to the data records in the same file — `_write_record` at line 88 calls `os.fsync(self.active_file.fileno())` when `sync_writes` is true. The hint file gets no such treatment.

In `log-structured-hash-table/bitcask.py`, the hint format is even simpler — `HINT_ENTRY_FMT = "!II"` (key_size, offset) at line 13 — and there's no CRC field at all. The data records use CRC32 (`HEADER_FMT = "!III"` — crc32, key_size, value_size at line 10), but hint entries carry zero integrity protection.

## The Crash Window During Compaction

Compaction follows this sequence:

1. Write a new merged data file containing only live keys
2. Write a hint file for that merged file
3. Delete old segment files
4. Update the in-memory index

If the process crashes **during step 2**, you get a partially written hint file on disk. The data file from step 1 may be complete (or may also be partial — it also lacks `fsync`), but the hint file is truncated mid-write.

## What Happens on Restart With a Partial Hint File

Recovery in both implementations **trusts the hint file unconditionally** when it exists:

**`hash-index-storage/bitcask.py:109-113`** (`_rebuild_index`):
```python
for fid in sorted(file_ids):
    hint_path = self._hint_path(fid)
    if os.path.exists(hint_path):
        self._load_hint_file(fid)       # <-- takes hint file's word for it
    else:
        self._scan_data_file(fid)        # <-- only scans if no hint exists
```

**`log-structured-hash-table/bitcask.py:82-86`** (`_recover`):
```python
for seg_id, seg_path in segments:
    hint_path = self._hint_path(seg_path)
    if os.path.exists(hint_path):
        self._load_hint_file(hint_path, seg_path)   # trusts hint
    else:
        self._scan_segment(seg_path)                 # only if no hint
```

The decision is binary: hint file exists → use it; doesn't exist → scan the data file. There's no validation that the hint file is complete or uncorrupted.

## How _load_hint_file Handles Truncation

Both implementations do handle mid-record truncation gracefully — they stop reading when they hit a short read:

**`log-structured-hash-table/bitcask.py:108-118`**:
```python
def _load_hint_file(self, hint_path, seg_path):
    with open(hint_path, "rb") as f:
        while True:
            header = f.read(HINT_HEADER_SIZE)
            if len(header) < HINT_HEADER_SIZE:
                break                        # stops on partial header
            key_size, offset = struct.unpack(HINT_ENTRY_FMT, header)
            key_bytes = f.read(key_size)
            if len(key_bytes) < key_size:
                break                        # stops on partial key
```

So a truncated hint file doesn't crash the process — it just **silently stops early**. Every key that was supposed to appear after the truncation point is simply missing from the recovered index. The data for those keys still exists in the data file, but recovery never reads the data file because the hint file's mere presence suppresses the scan.

## The Silent Data Loss Scenario

Here's the concrete sequence:

1. Store has keys A through Z across several segments
2. Compaction starts, writes keys A–Z into a new merged data file
3. Hint file write begins: A, B, C, D, E, F… crash at G
4. On restart, hint file exists → `_load_hint_file` runs, loads A–G
5. Keys H–Z are **gone from the index** — they exist in the data file but are unreachable
6. `get("H")` returns `None`. No error, no warning. The data is on disk but orphaned.

This is **silent key loss**, not corruption — the store appears healthy, just with fewer keys. It's the worst kind of failure because it passes every health check you'd think to run.

## Contrast With Data Records

The data records in `log-structured-hash-table/bitcask.py` have CRC32 checksums (lines 95-100 in `_scan_segment`):

```python
expected_crc = zlib.crc32(payload) & 0xFFFFFFFF
if crc != expected_crc:
    break  # corrupted record at end, stop scanning
```

If `_scan_segment` were used instead of `_load_hint_file`, a truncated or corrupted data file would trigger CRC failure and stop cleanly. But the hint file bypass means this protection is never invoked for compacted segments that have a hint file.

## What a Fix Would Look Like

Production Bitcask (Riak) solves this with:

1. **Hint file CRC**: each hint entry includes a checksum, so partial writes are detected
2. **`fsync` the hint file** before deleting old segments — ensures the hint is durable before the data it replaces is removed
3. **Atomic rename**: write to a temp file, `fsync`, rename — so the hint is either fully present or absent, never partial
4. **Fallback**: if any hint entry fails validation, discard the entire hint file and fall back to a full data scan

The simplest hardening for these implementations would be option 4: wrap `_load_hint_file` in a try/except and fall back to `_scan_segment` if anything looks wrong — or add a trailer checksum that covers the whole file, and if it's missing, scan instead.

## Topics to Explore

- [function] `log-structured-hash-table/bitcask.py:compact` — The compaction method itself (lines 200+, not in the observations) — need to see the exact ordering of data write, hint write, and old file deletion to map the full crash window
- [function] `hash-index-storage/bitcask.py:compact` — Same for the other implementation (lines 200+) — does it delete old segments before or after the hint file write?
- [general] `fsync-on-directory` — Even with `fsync` on the hint file, the directory entry must also be fsynced (`os.fsync(os.open(dir, os.O_RDONLY))`) for the rename to be durable on Linux — explore whether the data file writes have the same gap
- [file] `write-ahead-log/wal.py` — The WAL implementation uses `fsync` at lines 115, 128, 133, 184, 209, 261 — compare its durability discipline with the Bitcask hint file path to understand what "doing it right" looks like in this codebase
- [general] `tombstone-handling-in-hint-files` — The `log-structured-hash-table` hint format has no way to represent tombstones (it's just key_size + offset), so deleted keys from compaction may also have subtle semantics worth tracing

## Beliefs

- `hint-file-no-fsync` — Neither Bitcask implementation calls `fsync` on hint files, while `hash-index-storage/bitcask.py` does call `os.fsync` on data file writes at line 88
- `hint-file-no-checksum` — Hint file entries in both implementations lack checksums, unlike data records which use CRC32 (`log-structured-hash-table` line 10, `hash-index-storage` uses timestamps but no CRC)
- `hint-presence-suppresses-scan` — Both `_recover`/`_rebuild_index` methods use hint file existence as a binary gate: if the file exists at all, the data file is never scanned for that segment
- `partial-hint-silently-truncates` — `_load_hint_file` in both implementations treats a short read as end-of-file (not an error), silently dropping any keys that would have appeared after the truncation point
- `orphaned-data-unreachable` — Keys whose hint entries are lost to truncation remain in the data file on disk but are permanently unreachable through the index, with no error surfaced to the caller

