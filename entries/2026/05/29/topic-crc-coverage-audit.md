# Topic: Audit all struct.pack/CRC sites across the codebase (87 matches found) to identify which components protect only payload vs. full-record integrity

**Date:** 2026-05-29
**Time:** 08:16

# Integrity Protection Audit: Payload-Only vs. Full-Record CRC

This codebase has six distinct storage components, and they fall into three tiers of integrity protection: **payload-only CRC**, **data-only CRC (excluding WAL entry headers)**, and **no CRC at all**. None of them protect full records end-to-end.

## Components With CRC Protection

### 1. Log-Structured Hash Table (Bitcask) — Payload-only CRC

**File:** `log-structured-hash-table/bitcask.py`

The header format at line 10 is `"!III"` — three unsigned ints: `crc32, key_size, value_size`. The CRC is computed over `key_bytes + value` (the payload), but **not** over `key_size` or `value_size`:

- **Write path** (line 142): `crc = zlib.crc32(payload) & 0xFFFFFFFF` where `payload = key_bytes + value`
- **Read path** (`_scan_segment`, line 99): `expected_crc = zlib.crc32(payload) & 0xFFFFFFFF`
- **Point read** (`get`, line 182): same check

**What's unprotected:** If `key_size` or `value_size` in the header gets corrupted, the reader will slice the payload at wrong boundaries. The CRC will likely fail (because the wrong bytes get hashed), but this is an accidental detection, not a guarantee. A `key_size` bit-flip that happens to produce a valid CRC over the mis-sliced payload would silently return wrong data.

### 2. Write-Ahead Log — Partial-record CRC

**File:** `write-ahead-log/wal.py`

The CRC input at line 30 is `struct.pack("B", op_type_byte) + key + value` — it covers the operation type and the key/value payload. The header packed at line 33 is `"<IIQBi"`: `record_length, crc, seq_num, op_type_byte, len(key)`.

**What's protected:** `op_type_byte`, `key`, `value`
**What's unprotected:** `record_length`, `seq_num`, `len(key)`, and `len(value)` (packed separately at line 34). A corrupted `seq_num` would be accepted silently. A corrupted `record_length` would cause the framing loop to read the wrong number of bytes for `record_data`, likely producing a short read or garbage — but the failure mode is a crash, not a clean error.

Note that `op_type_byte` is redundantly present in both the CRC input and the header. It's the only header field that gets integrity protection.

### 3. B-Tree WAL — Data-only CRC (header excluded)

**File:** `b-tree-storage-engine/btree.py`

The WAL entry format (line 120) is: `seq(4B) + page_num(4B) + data_len(4B) + data + checksum(4B)`. The checksum at line 133 covers only `page_data`:

```
checksum = struct.pack('>I', self._checksum(page_data))
```

**What's protected:** The page data bytes written to the WAL.
**What's unprotected:** `seq`, `page_num`, and `data_len`. During recovery (line 162–164), a corrupted `page_num` would cause the recovered page to be written to the wrong location in the B-tree file — a silent, catastrophic corruption. The checksum would pass because the page data itself is fine; only the routing metadata is wrong.

## Components With No CRC

### 4. LSM Tree — No integrity checks

**File:** `log-structured-merge-tree/lsm.py`

Both the WAL class (lines 20–25) and SSTable class (lines 85–99) use bare length-prefixed records: `struct.pack(">I", len(k))` followed by the key bytes, then `struct.pack(">I", len(value))` followed by value bytes. No checksums anywhere. A single bit-flip in a length field causes cascading misframing of all subsequent records.

### 5. SSTable (standalone) — No integrity checks

**File:** `sstable-and-compaction/sstable.py`

Has a magic number (`SSTB`, line 50) for file type validation but no per-entry or per-file checksums. Entry format is `[key_len:2][key][timestamp:8][tombstone|value_len:4+value]`. Corruption in any length field silently misframes the scan.

### 6. Hash-Index Bitcask Store — No integrity checks

**File:** `hash-index-storage/bitcask.py`

Header format at line 10 is `"<dII"` — `timestamp, key_size, value_size`. No CRC field at all. Records are trusted blindly on read (line 97–100).

### 7. Bloom Filter — No integrity checks

**File:** `bloom-filter/bloom_filter.py`

Line 85 packs `m, k, count` as a header. The bit array is serialized after. No checksum protects either the header or the filter data.

## Summary Matrix

| Component | CRC? | Covers | Header protected? | Silent corruption risk |
|-----------|------|--------|--------------------|----------------------|
| `log-structured-hash-table/bitcask.py` | Yes | key + value | No (`key_size`, `value_size` excluded) | Mis-sliced payload |
| `write-ahead-log/wal.py` | Yes | op_type + key + value | Partial (`op_type` only) | Wrong `seq_num`, bad framing |
| `b-tree-storage-engine/btree.py` WAL | Yes | page data | No (`seq`, `page_num`, `data_len` excluded) | Page written to wrong location |
| `log-structured-merge-tree/lsm.py` | No | — | — | Cascading misframe |
| `sstable-and-compaction/sstable.py` | No | — | — | Cascading misframe |
| `hash-index-storage/bitcask.py` | No | — | — | Silent wrong reads |
| `bloom-filter/bloom_filter.py` | No | — | — | False negatives/positives |

The key finding: **no component in this codebase protects full-record integrity**. The three that have CRC all exclude framing/routing metadata from the checksum, which means header corruption can cause silent data loss or misrouting even when the payload checksum passes.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:WAL.recover` — The most dangerous gap: a corrupted `page_num` passes the checksum and writes a valid page to the wrong disk location
- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — How CRC failure during recovery is handled (stops scanning, potentially losing valid trailing records)
- [general] `length-prefix-framing-resilience` — Whether any component can resync after a corrupted length field, or if all suffer cascading misframes
- [file] `hash-index-storage/bitcask.py` — Compare this CRC-less Bitcask variant against the CRC-protected one in `log-structured-hash-table/` to understand when integrity was deemed necessary
- [general] `sstable-magic-number-vs-crc` — The SSTable uses a magic number for file-type validation but no per-entry checksums; explore whether the magic check gives any meaningful corruption protection beyond "wrong file type"

## Beliefs

- `bitcask-crc-excludes-header` — In `log-structured-hash-table/bitcask.py`, the CRC32 covers only the key+value payload; the `key_size` and `value_size` header fields are outside the checksum boundary
- `wal-crc-excludes-seq-and-lengths` — In `write-ahead-log/wal.py`, the CRC covers `op_type + key + value` but not `record_length`, `seq_num`, `key_len`, or `value_len`, so a corrupted sequence number is accepted silently
- `btree-wal-checksum-excludes-page-num` — In `b-tree-storage-engine/btree.py`, the WAL checksum covers only `page_data`, meaning a corrupted `page_num` during recovery writes a valid page to the wrong location without detection
- `lsm-and-sstable-have-no-checksums` — Neither `log-structured-merge-tree/lsm.py` nor `sstable-and-compaction/sstable.py` compute or verify any checksums; a single bit-flip in a length-prefix field causes cascading misframing of all subsequent records
- `no-component-has-full-record-crc` — Across all six storage components, none computes a CRC over the complete record (header + payload); integrity protection, where it exists, always excludes framing or routing metadata

