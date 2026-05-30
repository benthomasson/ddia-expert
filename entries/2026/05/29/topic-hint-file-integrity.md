# Topic: Neither implementation checksums hint files; explore what happens if a hint file is corrupted vs. simply missing (the fallback is a full data scan)

**Date:** 2026-05-29
**Time:** 11:56

# Hint File Corruption vs. Missing: The Unchecksummed Fast Path

## The Two Implementations

Both Bitcask implementations have the same architectural pattern: hint files are an **optimization** for startup. They store pre-computed index entries so the engine can skip scanning raw data files. The recovery logic in both follows the same branch:

**`hash-index-storage/bitcask.py:109-114`** — `_rebuild_index`:
```python
if os.path.exists(hint_path):
    self._load_hint_file(fid)
else:
    self._scan_data_file(fid)
```

**`log-structured-hash-table/bitcask.py:80-84`** — `_recover`:
```python
if os.path.exists(hint_path):
    self._load_hint_file(hint_path, seg_path)
else:
    self._scan_segment(seg_path)
```

## What Happens When a Hint File Is Missing

This is the safe case. Both implementations fall back to scanning the raw data file:

- **`hash-index-storage/bitcask.py:117`** calls `_scan_data_file`, which walks the data file record-by-record, reading actual keys, offsets, and sizes. This implementation has no CRC on data records either, but the index it rebuilds is at least consistent with what's on disk.

- **`log-structured-hash-table/bitcask.py:89`** calls `_scan_segment`, which walks the segment and **verifies CRC32 on every record** (lines 99–100). Corrupted records at the tail are caught, and scanning stops cleanly at the break on line 100. The rebuilt index only contains verified entries.

The cost is time — a full sequential read of the data file instead of reading the smaller hint file. But correctness is preserved.

## What Happens When a Hint File Is Corrupted

This is where things break silently.

### `hash-index-storage/bitcask.py:139-150` — `_load_hint_file`

The loader reads the hint file as a flat byte buffer and parses entries in a loop using `struct.unpack`. There is no checksum, no magic number, no consistency check. A corrupted hint file produces one of two outcomes:

1. **Structural corruption** (garbled sizes): `struct.unpack` reads a bogus `key_size`, tries to slice past the buffer, and raises an exception. The engine crashes on startup — there's no try/except wrapping this path.

2. **Subtle corruption** (flipped bits in offset or size fields): The parsing succeeds, but `keydir` is populated with wrong offsets. A subsequent `get()` call reads garbage from the data file. The `assert read_key == key` on line 185 *might* catch this if the corrupted offset lands on a different record, but if the offset is just slightly wrong (pointing mid-record), the decode will produce garbage or raise a `struct` error at read time, not at startup.

### `log-structured-hash-table/bitcask.py:107-117` — `_load_hint_file`

Same pattern, same vulnerability. The hint entry format (`HINT_ENTRY_FMT = "!II"` — key_size and offset) is parsed without any integrity check. But here, the asymmetry is worse: **data records are CRC32-protected** (lines 95–100, 178–185), yet hint files bypass that protection entirely.

When `get()` later reads a record at a corrupted offset, the CRC check on line 182–185 *will* catch the mismatch and raise `CorruptionError`. So the corruption is detected — but at read time, not at startup, and only for the specific keys whose hint entries were damaged. The engine appears healthy until a user hits one of those keys.

## The Asymmetry That Matters

| Scenario | `hash-index-storage` | `log-structured-hash-table` |
|---|---|---|
| Hint missing | Full data scan (no CRC) | Full data scan (with CRC) |
| Hint corrupted (structural) | Crash on startup | Crash on startup |
| Hint corrupted (subtle) | Silent wrong reads, `assert` may catch | Silent bad index, `CorruptionError` on `get()` |
| Data file corrupted | No detection | CRC catches it |

The critical gap: **neither implementation can distinguish between a valid hint file and a corrupted one at load time**. A missing hint file is strictly safer than a corrupted one, because the fallback path (full scan) is the one that actually verifies data integrity.

In production Bitcask (Riak's implementation), hint files include per-entry CRC32 checksums, and a corrupted hint file triggers a fallback to the data scan. These reference implementations skip that step, which means a single bit flip in a hint file can silently poison the in-memory index.

## Topics to Explore

- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — Compare the CRC verification logic here with the hint file loader to see exactly what protection is bypassed
- [function] `hash-index-storage/bitcask.py:_write_hint_file` — Understanding what gets written helps you see what a checksum would protect
- [function] `log-structured-hash-table/bitcask.py:get` — Trace the read path to see how a poisoned index entry eventually surfaces as a `CorruptionError`
- [general] `hint-file-checksum-addition` — Design what adding CRC32 to hint entries would look like, and whether the fallback-to-scan should be automatic or raise
- [file] `b-tree-storage-engine/btree.py` — Compare the WAL's checksum-and-skip-on-corruption strategy (line 162–164) as a model for how hint files could handle integrity

## Beliefs

- `hint-file-no-integrity-check` — Neither Bitcask implementation validates hint file integrity on load; corrupted hint data is trusted and written directly into the in-memory index
- `missing-hint-safe-fallback` — When a hint file is absent, both implementations fall back to `_scan_data_file`/`_scan_segment`, which rebuild the index from authoritative data records
- `log-structured-crc-bypassed-by-hints` — `log-structured-hash-table/bitcask.py` checksums every data record with CRC32, but loading from a hint file skips all CRC verification, creating an integrity gap
- `subtle-corruption-deferred-detection` — In `log-structured-hash-table`, a subtly corrupted hint file produces no error until a `get()` hits a poisoned entry, at which point CRC mismatch raises `CorruptionError`
- `hash-index-assert-partial-guard` — `hash-index-storage/bitcask.py:185` asserts `read_key == key` on every `get()`, which partially detects index corruption but only when the wrong offset lands on a parseable record with a different key

