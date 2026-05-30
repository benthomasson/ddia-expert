# Topic: Production WALs (e.g., RocksDB, LevelDB) include CRC32 checksums per record to detect corruption, not just truncation. This implementation silently accepts corrupted records as long as the length fields parse.

**Date:** 2026-05-29
**Time:** 11:42

## The Claim Is Wrong — This Implementation Does Validate Checksums

The belief states that this WAL "silently accepts corrupted records as long as the length fields parse." That's incorrect. Looking at the code, CRC32 validation is present and functional — but it has a subtle coverage gap.

### What the code actually does

**Write path** (`_encode_record`, `wal.py:29-34`): Each record gets a CRC32 computed over `op_type_byte + key + value` and stored in the record header:

```python
crc_data = struct.pack("B", op_type_byte) + key + value
crc = zlib.crc32(crc_data) & 0xFFFFFFFF
```

**Read path** (`_read_record`, `wal.py:37-58`): The CRC is recomputed and checked on every read. A mismatch raises `ValueError`:

```python
crc_data = struct.pack("B", op_type_byte) + key + value
expected_crc = zlib.crc32(crc_data) & 0xFFFFFFFF
if crc != expected_crc:
    raise ValueError(f"CRC mismatch at seq {seq_num}")
```

**Test coverage** (`test_wal.py:47-57`): `test_corruption` explicitly corrupts the last 5 bytes of a WAL file and verifies that only the uncorrupted first record survives replay. The corrupted second record is detected and dropped.

### What the CRC does NOT cover

The real nuance — and likely what originally motivated this belief — is that the CRC input is `op_type + key + value`. Two fields are excluded:

1. **`seq_num`** — A bit-flip in the sequence number passes CRC validation silently. This matters: sequence numbers drive replay ordering, deduplication, and truncation boundaries. A corrupted seq_num could cause a record to be replayed out of order or skipped by `truncate(up_to_seq)`.

2. **Length fields** (`record_length`, `key_len`, `val_len`) — These aren't directly in the CRC input. However, corrupted lengths cause the wrong bytes to be extracted as key/value, which almost always triggers a CRC mismatch indirectly. The exception: if `record_length` corrupts to a *smaller* value, `_read_record` returns `None` (treating it as truncation at line 43-44) rather than raising a CRC error — the corruption looks like a short read.

### How production WALs differ

RocksDB's WAL format checksums the **entire record payload** — type byte, data, and a record-type tag — in a per-block framing that also detects block-boundary corruption. LevelDB similarly CRCs the type + data + length. The key design difference: production WALs treat the checksum as covering all mutable fields, so no field can silently corrupt.

### Recovery behavior

When `_read_record` raises `ValueError`, callers in `_recover_seq_num` (line 87-92) and `replay`/`iterate` treat it as end-of-valid-data: they `break` out of the read loop. This means a single corrupted record in the middle of a WAL file causes all subsequent valid records in that file to be silently discarded — a correct-but-conservative strategy that production systems also use for the tail of a WAL, but not for mid-file corruption (RocksDB can skip corrupted blocks and recover later records).

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_record` — Trace exactly which byte ranges the CRC covers vs. which it doesn't, and construct a corruption scenario where seq_num flips but CRC passes
- [function] `write-ahead-log/wal.py:_recover_seq_num` — Understand how a corrupted seq_num during recovery could set the wrong starting sequence for new writes
- [general] `rocksdb-wal-block-format` — Compare RocksDB's fixed-size block framing with this implementation's variable-length record framing to understand why block-based formats recover more records after mid-file corruption
- [file] `b-tree-storage-engine/btree.py` — The B-tree's WAL (line 120-176) uses a similar CRC pattern but covers `page_data` entirely — compare the two approaches
- [file] `log-structured-hash-table/bitcask.py` — Bitcask's CRC covers `key + value` payload (lines 95-100, 142-143) — same gap pattern as the WAL

## Beliefs

- `wal-has-crc32-validation` — `_read_record` computes and validates CRC32 on every record read, raising `ValueError` on mismatch; the claim that corruption is silently accepted is false
- `wal-crc-excludes-seq-num` — The CRC input is `op_type_byte + key + value`; `seq_num` is not checksummed, so a bit-flip in the sequence number is undetectable
- `wal-corruption-discards-tail` — When a CRC mismatch is hit during recovery or replay, all subsequent records in that WAL file are silently skipped rather than individually validated
- `wal-length-corruption-mimics-truncation` — If `record_length` corrupts to a smaller value, `_read_record` returns `None` (EOF/truncation) instead of raising a CRC error, masking corruption as a clean short file

