# Topic: What integrity scheme the WAL uses (CRC32, SHA, length prefix) — critical for understanding `test_corruption`'s 5-byte overwrite strategy

**Date:** 2026-05-28
**Time:** 18:47

## WAL Integrity Scheme: CRC32 + Length Prefix

The WAL uses a **two-layer integrity scheme**: a 4-byte length prefix for framing and a CRC32 checksum for content verification. Understanding both layers is key to seeing why `test_corruption`'s 5-byte overwrite reliably detects corruption.

### Record Wire Format

`_encode_record` (`wal.py:29-35`) builds each record as:

```
[record_length: u32][crc: u32][seq_num: u64][op_type: u8][key_len: i32][key][val_len: i32][value]
 ← 4 bytes →         ← 4 →    ← 8 →        ← 1 →       ← 4 →              ← 4 →
```

The **length prefix** (`record_length`, first 4 bytes) tells the reader how many bytes to consume *after* the prefix itself. This is computed at line 33:

```python
record_length = 4 + 8 + 1 + 4 + len(key) + 4 + len(value)  # 21 + key + value
```

Note: the length prefix counts the payload including CRC but *not* itself — `_read_record` reads 4 bytes for the length, then reads `record_length` more bytes.

### CRC32 Coverage

The CRC32 (via `zlib.crc32`, line 31) covers only the **semantic content**: `op_type_byte + key + value`. It deliberately excludes `seq_num`, `key_len`, and `val_len` from the checksum. This means:

- Corruption to key/value data → CRC mismatch → `ValueError` raised at line 53
- Corruption to the length prefix → short/wrong read → returns `None` (treated as EOF/truncation)
- Corruption to `seq_num` → **undetected** (silent wrong ordering)

### Why the 5-Byte Overwrite Works

In `test_corruption` (`test_wal.py:48-58`):

```python
f.seek(-5, 2)                              # 5 bytes before end of file
f.write(b"\xff\xff\xff\xff\xff")            # overwrite with 0xFF
```

Two records are written: `PUT "a" "1"` and `PUT "b" "2"`. For these short key/value pairs, each record is roughly 25–27 bytes total (4-byte length prefix + 21 bytes header + tiny key/value). The 5-byte overwrite from the end lands squarely in the **second record's value or trailing bytes**.

This triggers detection through one of two paths:

1. **CRC mismatch**: If the corruption hits the value bytes, `zlib.crc32` of the corrupted content won't match the stored CRC → `ValueError` at line 53 → `_recover_seq_num` breaks out of the read loop (line 90), and `replay` stops, yielding only the first record.

2. **Length/framing corruption**: If the corruption hits a length field, `_read_record` reads the wrong number of bytes → either a short read (returns `None`) or misaligned parsing that cascades into a CRC failure.

Either way, the result is the same: `replay()` returns only the first record (`len(records) == 1`), proving the WAL correctly stops at the corruption boundary rather than returning garbage.

### Recovery Philosophy

The WAL follows a **fail-fast, truncate-at-corruption** strategy. In `_recover_seq_num` (line 83), a `ValueError` from CRC mismatch causes the scanner to `break` — it does not skip corrupted records to find valid ones later in the file. This is the standard approach for sequential WALs: everything after corruption is untrusted.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_record` — The deserialization path where CRC validation and framing errors are detected; trace it byte-by-byte to understand failure modes
- [function] `write-ahead-log/wal.py:_encode_record` — Compare what the CRC covers (op+key+value) vs. what it excludes (seq_num, lengths) — this is a deliberate design choice with tradeoffs
- [general] `wal-recovery-semantics` — The WAL stops at the first corrupted record rather than scanning past it; explore whether this is always safe (what if corruption hits the middle of a multi-file WAL?)
- [file] `write-ahead-log/test_wal.py` — The truncation and rotation tests reveal how the WAL handles multi-file scenarios, which interact with the corruption-stops-reading behavior
- [general] `crc32-vs-cryptographic-hash` — CRC32 detects accidental corruption but not adversarial tampering; consider when a stronger integrity check (SHA-256) would be warranted

## Beliefs

- `wal-uses-crc32-not-sha` — WAL integrity uses `zlib.crc32` (32-bit, non-cryptographic); it detects accidental corruption but not intentional tampering
- `crc32-covers-op-key-value-only` — The CRC32 checksum covers `op_type + key + value` but excludes `seq_num`, `key_len`, and `val_len`, meaning sequence number corruption is silent
- `wal-stops-at-first-corruption` — Recovery (`_recover_seq_num`, `replay`) breaks on the first CRC error or short read; records after a corrupted record are discarded even if intact
- `record-format-is-length-prefixed` — Each record starts with a 4-byte little-endian length prefix that does not count itself, enabling the reader to frame records without delimiters
- `corruption-test-exploits-tail-position` — `test_corruption` overwrites the last 5 bytes of the file, which targets the second record's payload, guaranteeing a CRC mismatch while leaving the first record intact

