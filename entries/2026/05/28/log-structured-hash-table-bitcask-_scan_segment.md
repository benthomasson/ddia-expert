# Function: _scan_segment in log-structured-hash-table/bitcask.py

**Date:** 2026-05-28
**Time:** 19:04

## `_scan_segment` — Segment File Scanner for Index Recovery

### Purpose

`_scan_segment` rebuilds the in-memory hash index by sequentially reading every record from a single segment file on disk. It's the crash-recovery path: when the store starts up (or when `rebuild_index` is called explicitly) and no hint file exists for a segment, this method replays the segment's write log to reconstruct which keys live where.

This is the slow path. The fast path is `_load_hint_file`, which reads a pre-built index. `_scan_segment` exists because hint files are optional — they're only created for frozen (compacted) segments, not the active one. On first startup after a crash, the active segment has no hint file, so the store must scan it record by record.

### Contract

**Preconditions:**
- `seg_path` must point to a valid, readable file on disk containing zero or more Bitcask records in the binary format `[crc32 | key_size | value_size | key_bytes | value_bytes]`.
- `self._index` must already exist (initialized by `__init__`).
- Segments must be scanned **oldest-to-newest** for the index to be correct, since later writes for the same key must overwrite earlier ones. The caller (`_recover`) guarantees this ordering.

**Postconditions:**
- Every intact, non-tombstoned record in the segment has its key mapped in `self._index` to `(seg_path, byte_offset)`.
- Every tombstoned key has been **removed** from `self._index` entirely — including entries that may have been placed there by scanning earlier segments.
- The file is closed (opened in a `with` block).

**Invariants:**
- The index always reflects the latest write for each key across all segments scanned so far. This is the append-only log semantics: the last write wins.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `self` | `BitcaskStore` | The store instance; `self._index` is mutated |
| `seg_path` | `str` | Absolute path to a `.dat` segment file |

No validation is performed on `seg_path` — if the file doesn't exist, the `open()` call raises `FileNotFoundError`. This is intentional: the callers (`_recover`, `rebuild_index`) only pass paths discovered by `_find_existing_segments`.

### Return Value

`None`. The method communicates its results entirely through mutation of `self._index`.

### Algorithm

1. **Open the segment** for binary reading.
2. **Loop** until the file is exhausted or a corrupt/incomplete record is found:
   - **Record the current offset** via `f.tell()` — this is the byte position that will go into the index.
   - **Read the fixed-size header** (12 bytes: three unsigned 32-bit ints in network byte order). If fewer than 12 bytes remain, the file is done.
   - **Unpack** the header into `crc`, `key_size`, `value_size`.
   - **Read the payload** (`key_size + value_size` bytes). If the read is short, the record was partially written (crash mid-write); stop scanning.
   - **Verify integrity**: compute CRC32 over the payload and compare to the stored CRC. If they disagree, the record is corrupt; stop scanning. Notably, corruption terminates the scan entirely — it does not skip the bad record and continue, because in an append-only log, everything after a corrupt record is suspect.
   - **Decode the key** from the first `key_size` bytes of the payload (UTF-8).
   - **Extract the value** from the remaining bytes.
   - **Handle tombstones**: if the value matches the sentinel `b"__BITCASK_TOMBSTONE__"`, remove the key from the index. Otherwise, insert/overwrite the index entry with `(seg_path, offset)`.

### Side Effects

- **Mutates `self._index`**: adds, updates, or removes entries. This is the whole point.
- **Disk I/O**: reads the entire segment file sequentially. For large segments this is the dominant cost at startup.
- **No writes**: this method is read-only with respect to disk. It never modifies the segment file.

### Error Handling

The method **swallows all data integrity errors silently** by breaking out of the loop:

- **Short header read**: break (EOF or partial write)
- **Short payload read**: break (partial write)
- **CRC mismatch**: break (corruption)

No exception is raised, no warning is logged. The design philosophy is that a partial or corrupt record at the tail of a segment is expected after a crash — the process died mid-write. Everything before the bad record is assumed valid. This is safe because writes are append-only and flushed after each record (`_write_record` calls `flush()`).

However, if `seg_path` doesn't exist or isn't readable, `open()` raises a standard OS exception that propagates to the caller.

### Usage Patterns

Called in two contexts:

1. **Startup recovery** (`_recover`): scans each segment that lacks a hint file, in segment-ID order.
2. **Manual rebuild** (`rebuild_index`): called after `self._index.clear()` to reconstruct from scratch.

**Caller obligation**: segments must be scanned in chronological order. If you scan segment 3 before segment 1, you'll get the wrong index — an older value for a key could overwrite the newer one.

### Dependencies

| Dependency | Usage |
|------------|-------|
| `struct` (stdlib) | Unpacking the binary header format `!III` |
| `zlib` (stdlib) | CRC32 checksum computation |
| `HEADER_FMT`, `HEADER_SIZE` | Module-level constants defining the record header layout |
| `TOMBSTONE` | Sentinel value `b"__BITCASK_TOMBSTONE__"` for deletion markers |

### Assumptions Not Enforced by Types

1. **Segment ordering is the caller's responsibility.** Nothing prevents calling `_scan_segment` out of order, which would silently corrupt the index.
2. **Keys are valid UTF-8.** The `.decode("utf-8")` will raise `UnicodeDecodeError` if a key contains invalid bytes — this error is not caught and will propagate, aborting recovery.
3. **Tombstone values are never legitimately stored.** If a user calls `put(key, b"__BITCASK_TOMBSTONE__")`, the key becomes invisible. There's no escaping mechanism.
4. **Corruption only occurs at the tail.** The break-on-CRC-mismatch strategy assumes corruption doesn't happen mid-file with valid records after it. A bit flip in the middle of a segment would cause all subsequent valid records to be silently dropped from the index.

---

## Topics to Explore

- [function] `log-structured-hash-table/bitcask.py:_load_hint_file` — The fast-path alternative to scanning; compare what information hint files omit (tombstones, values) and the trade-offs
- [function] `log-structured-hash-table/bitcask.py:compact` — How compaction rewrites frozen segments and why the scan order invariant is re-established after compaction
- [function] `log-structured-hash-table/bitcask.py:_write_record` — The write side of the record format that `_scan_segment` reads; verify the symmetry between writer and reader
- [general] `bitcask-crash-recovery-semantics` — How the break-on-corruption strategy interacts with `flush()` guarantees and OS write atomicity — is a partial header possible, and what happens if `fsync` is not called?
- [file] `log-structured-hash-table/test_bitcask.py` — Test coverage for crash recovery, corruption scenarios, and tombstone handling during index rebuild

---

## Beliefs

- `scan-segment-ordering-invariant` — `_scan_segment` must be called on segments in ascending segment-ID order for the index to correctly reflect last-write-wins semantics; no runtime check enforces this.
- `corruption-terminates-scan` — A CRC mismatch or short read stops scanning the entire segment; records after the corrupt point are silently lost from the index, even if they are individually valid.
- `tombstone-removes-cross-segment` — A tombstone record in segment N removes the key from the index even if the key's live value was written in an earlier segment, because the index is a flat dict shared across all scans.
- `scan-segment-is-read-only` — `_scan_segment` never modifies any file on disk; its only side effect is mutating `self._index`.
- `utf8-decode-unguarded` — If a segment contains a key with invalid UTF-8 bytes, `_scan_segment` raises an unhandled `UnicodeDecodeError`, aborting recovery for all subsequent segments.

