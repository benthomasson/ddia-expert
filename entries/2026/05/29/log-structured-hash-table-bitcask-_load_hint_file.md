# Function: _load_hint_file in log-structured-hash-table/bitcask.py

**Date:** 2026-05-29
**Time:** 07:26

## `_load_hint_file` — Fast Index Recovery from Precomputed Summaries

### Purpose

`_load_hint_file` reconstructs the in-memory hash index from a **hint file** instead of scanning the full segment data file. This is a startup optimization straight from the Bitcask paper: hint files contain only the keys and their offsets, stripping out the values entirely. For a segment with large values, this can reduce recovery I/O by orders of magnitude — you read kilobytes of hints instead of megabytes of data.

It exists because the alternative — `_scan_segment` — must read every byte of every record (key + value + CRC) just to extract the key and offset. Hint files are a precomputed projection of exactly the information the index needs.

### Contract

**Preconditions:**
- `hint_path` points to an existing, readable file in the hint format (`[key_size: u32, offset: u32, key_bytes: var]` repeated)
- `seg_path` is the corresponding `.dat` segment file — the offsets stored in the hint file are byte positions within this segment
- The hint file only contains entries for **live** keys at the time it was written (tombstoned keys are excluded by `create_hint_files`)

**Postconditions:**
- `self._index` contains a `(seg_path, offset)` entry for every key found in the hint file
- Any prior index entry for the same key is overwritten (last-write-wins, which is correct because `_recover` processes segments oldest-to-newest)

**Invariant assumed but not enforced:** The hint file is consistent with the segment file. If the hint file is corrupt or out of sync, the index will contain garbage offsets that will cause `CorruptionError` on read.

### Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `hint_path` | `str` | Absolute path to the `.hint` file to read |
| `seg_path` | `str` | Absolute path to the corresponding `.dat` segment — not opened here, just stored as the index value |

Neither parameter is validated. A nonexistent `hint_path` raises `FileNotFoundError`. A wrong `seg_path` silently corrupts the index.

### Return Value

None. All output is via mutation of `self._index`.

### Algorithm

1. Open the hint file in binary read mode.
2. **Loop:** read a fixed-size header (`HINT_HEADER_SIZE` = 8 bytes, two `u32` network-order integers).
3. Unpack the header into `key_size` (length of the key in bytes) and `offset` (byte position of the full record in the segment file).
4. Read `key_size` bytes for the key. If a short read occurs, break — the file is truncated or done.
5. Decode the key from UTF-8 and insert `(seg_path, offset)` into `self._index`.
6. Repeat until a header read returns fewer than `HINT_HEADER_SIZE` bytes.

The critical thing: `offset` here is the offset **in the segment file**, not in the hint file. When `get()` later uses this offset, it seeks to that position in `seg_path` and reads the full record (header + key + value + CRC check).

### Side Effects

- **Mutates `self._index`** — overwrites entries for any keys already present. This is intentional: during recovery, segments are processed oldest-first, so newer segments' entries correctly shadow older ones.
- **No disk writes.** This is a pure read operation.
- **No file handle caching.** Uses a `with` block, so the hint file handle is closed on exit.

### Error Handling

Errors are handled via **silent truncation**, not exceptions:

- Short reads on the header or key bytes → `break` out of the loop. This treats any partial write at the tail as end-of-file rather than corruption.
- No CRC or checksum validation on hint file contents. A bit-flipped offset silently enters the index and will only surface as a `CorruptionError` when `get()` reads the segment and the CRC check fails.
- A malformed hint file (e.g., `key_size` is absurdly large) will cause a large allocation in `f.read(key_size)`, potentially an `OverflowError` or `MemoryError`. No bounds checking.

### Usage Patterns

Called from exactly two sites:

1. **`_recover()`** (line ~73) — during startup, for each segment that has a `.hint` file alongside it. Falls back to `_scan_segment` if no hint file exists.
2. **`rebuild_index()`** (line ~230) — explicit full index rebuild, same hint-or-scan logic.

In both cases, the caller iterates segments in sorted order (by segment ID), so last-write-wins semantics in the index produce the correct result.

The hint files themselves are created by `create_hint_files()`, which only writes entries for non-tombstoned keys whose index still points to that segment — so `_load_hint_file` never sees tombstones and unconditionally inserts every entry it reads.

### Dependencies

- **`struct`** — binary unpacking with `HINT_ENTRY_FMT = "!II"` (network byte order, two unsigned 32-bit ints)
- **`HINT_HEADER_SIZE`** — precomputed as `struct.calcsize(HINT_ENTRY_FMT)` = 8 bytes
- **`self._index`** — the in-memory `dict[str, tuple[str, int]]` that is the entire point of Bitcask's design

### Notable Assumptions

1. **Keys are valid UTF-8.** `key_bytes.decode("utf-8")` will raise `UnicodeDecodeError` on malformed data — this is the one failure mode that is not silently swallowed.
2. **Hint files are trustworthy.** No integrity check means a corrupted hint file is worse than a missing one — missing triggers a full scan with CRC checks, while a corrupted hint silently loads bad data.
3. **Offsets fit in 32 bits.** The `!II` format caps segment files at 4 GiB. This matches the `max_segment_size` default of 1 MiB, but nothing prevents a caller from setting a larger size.
4. **No tombstone entries in hint files.** `_load_hint_file` does not check for tombstones (unlike `_scan_segment`). This is correct only because `create_hint_files` filters them out at write time — the correctness of this reader depends on the writer's behavior.

---

## Topics to Explore

- [function] `log-structured-hash-table/bitcask.py:create_hint_files` — The writer half of the hint file contract; understanding what it filters and writes is essential to knowing why `_load_hint_file` can skip tombstone checks
- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — The slow-path alternative that reads full records with CRC validation; compare what guarantees you get vs. the hint path
- [function] `log-structured-hash-table/bitcask.py:_recover` — The orchestrator that decides hint-vs-scan per segment and controls processing order; the correctness of last-write-wins lives here
- [general] `bitcask-hint-file-format` — The original Bitcask paper's hint file design and how this implementation simplifies it (no timestamp, no value size)
- [function] `log-structured-hash-table/bitcask.py:compact` — Compaction deletes old segments and their hint files; understanding the lifecycle of hint files requires knowing when they're destroyed

## Beliefs

- `hint-file-no-integrity-check` — `_load_hint_file` performs no CRC or checksum validation on hint file data; a corrupted hint file silently loads bad offsets into the index
- `hint-file-no-tombstones` — Hint files never contain tombstone entries because `create_hint_files` filters them out at write time; `_load_hint_file` relies on this invariant without verifying it
- `hint-offset-is-segment-offset` — The `offset` value stored in a hint entry is a byte position in the corresponding `.dat` segment file, not a position within the hint file itself
- `recovery-order-correctness` — Index correctness during recovery depends on segments being processed in ascending ID order so that newer entries overwrite older ones; `_load_hint_file` itself does not enforce this ordering

