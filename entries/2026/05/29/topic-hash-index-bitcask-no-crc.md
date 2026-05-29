# Topic: Whether the absence of CRC in `hash-index-storage/bitcask.py` is a deliberate simplification or an oversight, and what happens on silent corruption

**Date:** 2026-05-29
**Time:** 08:13

## CRC Absence in `hash-index-storage/bitcask.py`: Deliberate Simplification

The absence of CRC is a **deliberate simplification**, not an oversight. The evidence across the codebase makes this clear through contrast.

### The two Bitcask implementations tell different stories

The repo contains two separate Bitcask implementations at different fidelity levels:

1. **`log-structured-hash-table/bitcask.py`** — includes full CRC32 integrity checking. The header format at line 8 is `"!III"` (crc32, key_size, value_size), it computes checksums via `zlib.crc32` in `_write_record` (line 148), validates on read in `get()` (lines 172–180), and defines a dedicated `CorruptionError` exception (line 18). During recovery in `_scan_segment` (line 96), a CRC mismatch halts scanning — treating corruption as an end-of-valid-data marker.

2. **`hash-index-storage/bitcask.py`** — has no CRC at all. Its header format at line 10 is `"<dII"` (timestamp, key_size, value_size). No `import zlib`, no checksum field, no integrity validation anywhere.

This isn't a case of someone forgetting one line. The entire integrity subsystem — the header field, the error class, the check-on-read path, the check-on-scan path — is absent. That's a design choice, not a missed detail.

### The repo stratifies implementations by concern

Other modules in the repo follow the same pattern of having CRC where integrity is the *point* of the module:

- **`write-ahead-log/wal.py`** (lines 30–31, 53–56): CRC32 on every record, raises `ValueError` on mismatch — because crash recovery correctness *is* the WAL's reason to exist.
- **`b-tree-storage-engine/btree.py`** (lines 120, 133, 162–164): Checksums on WAL entries for the B-tree's crash recovery path.

The `hash-index-storage/` module focuses on the **hash-index data structure** — the keydir, append-only log layout, compaction, hint files. CRC would be orthogonal to what it's demonstrating.

### What happens on silent corruption

Without CRC, silent corruption produces **wrong answers with no error signal**:

- **Corrupted value bytes**: `get()` (line 177) reads `size` bytes from the offset, unpacks the header, and returns whatever it finds. A bit-flip in the value region returns garbage as if it were the real value. The caller has no way to distinguish this from a valid read.

- **Corrupted key bytes**: The `assert read_key == key` at line 179 acts as an accidental partial integrity check — if the key portion is corrupted, this assertion fires. But this only catches corruption in the key region, not the value.

- **Corrupted header (key_size/val_size fields)**: This is the worst case. A corrupted `key_size` or `val_size` causes the code to slice the wrong number of bytes, potentially reading across record boundaries or triggering a `UnicodeDecodeError` on the `.decode("utf-8")` call (line 100). During `_scan_data_file` (line 113), this would derail the entire sequential scan, silently skipping or misinterpreting all subsequent records in that file.

- **Corrupted data during compaction**: `compact()` reads and rewrites records from immutable files. If a record is silently corrupted, the corrupted data gets faithfully copied into the compacted output — laundering the corruption into a clean-looking file.

The `log-structured-hash-table` variant handles all of these: `_scan_segment` (line 103) stops at the first CRC mismatch, and `get()` (line 178) raises `CorruptionError` rather than returning bad data.

### Summary

The `hash-index-storage/bitcask.py` is a teaching implementation focused on the keydir/append-log/compaction architecture from DDIA Chapter 3. CRC was intentionally left out to keep the record format and code paths simple. For a production-grade version with integrity checking, the repo provides `log-structured-hash-table/bitcask.py`.

---

## Topics to Explore

- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — Shows how CRC failures are used as an end-of-valid-data sentinel during crash recovery, a subtle design choice worth understanding
- [function] `hash-index-storage/bitcask.py:compact` — Compaction rewrites records without integrity checks; trace how corrupted data propagates through this path
- [file] `write-ahead-log/wal.py` — Compare its CRC strategy (reject on mismatch) with the log-structured variant (stop scanning on mismatch) — two valid approaches to the same problem
- [general] `hint-file-integrity` — Neither implementation checksums hint files; explore what happens if a hint file is corrupted vs. simply missing (the fallback is a full data scan)
- [function] `hash-index-storage/bitcask.py:_scan_data_file` — Walk through what a corrupted `key_size` field does to the sequential scan loop — it can't self-correct without record delimiters or checksums

---

## Beliefs

- `hash-index-no-crc-by-design` — `hash-index-storage/bitcask.py` intentionally omits CRC to focus on the keydir/append-log/compaction architecture; `log-structured-hash-table/bitcask.py` provides the integrity-checking variant
- `silent-corruption-returns-garbage` — A bit-flip in a value region of `hash-index-storage/bitcask.py` causes `get()` to return corrupted data with no error, since the only integrity check is the key-equality assertion at line 179
- `crc-mismatch-halts-scan` — In `log-structured-hash-table/bitcask.py:_scan_segment`, a CRC mismatch breaks out of the scan loop (line 103), treating all subsequent bytes in that segment as untrustworthy
- `compaction-propagates-corruption` — `hash-index-storage/bitcask.py:compact` copies records without validation, so silently corrupted data survives compaction into new files
- `header-corruption-derails-sequential-scan` — A corrupted `key_size` or `val_size` in `hash-index-storage/bitcask.py` causes `_scan_data_file` to read wrong byte boundaries, potentially misinterpreting all subsequent records in the file

