# Topic: Why production systems (RocksDB, PostgreSQL) prefer the Castagnoli polynomial over ISO 3309, and what Python library changes that would require

**Date:** 2026-05-29
**Time:** 11:57

# CRC-32 vs CRC-32C: Why Production Systems Choose Castagnoli

## What the codebase does today

Every module that checksums data uses `zlib.crc32` — the ISO 3309 polynomial (0xEDB88320). You can see this in:

- **WAL**: `write-ahead-log/wal.py:31` and `:54` — checksums each record on write, verifies on read
- **B-tree**: `b-tree-storage-engine/btree.py:176` — the `_checksum` function wraps `zlib.crc32(data) & 0xFFFFFFFF`
- **Bitcask**: `log-structured-hash-table/bitcask.py:99,142,182,275` — checksums the key+value payload in every record

The `& 0xFFFFFFFF` mask ensures an unsigned 32-bit result (Python 2 legacy; Python 3's `zlib.crc32` already returns unsigned, but the mask is harmless).

## Why production systems use CRC-32C instead

The ISO 3309 polynomial and the Castagnoli polynomial are both 32-bit CRCs with identical output width, but they differ in two critical ways:

### 1. Hardware acceleration

Intel added the `CRC32` instruction in SSE 4.2 (2008), and ARM added it in ARMv8. Both implement the **Castagnoli polynomial specifically** — not ISO 3309. This means CRC-32C runs at memory-bandwidth speed (multiple GB/s) while ISO 3309 CRC-32 must be computed in software lookup tables, typically 10-20x slower.

For a storage engine like RocksDB that checksums every block on read and write, or PostgreSQL's WAL that checksums every record (exactly like `write-ahead-log/wal.py:31`), this is the difference between checksumming being free and checksumming being a measurable fraction of I/O latency.

### 2. Better error detection properties

The Castagnoli polynomial has superior Hamming distance properties for the data sizes typical in storage systems (4KB-64KB blocks). It detects all 3-bit errors in messages up to 65,520 bits, vs. ISO 3309 which only guarantees that up to ~32,768 bits. For page-sized data — like the B-tree pages in `b-tree-storage-engine/btree.py:176` — this matters.

## What Python library changes it would require

Switching this codebase from ISO 3309 to Castagnoli requires replacing `zlib.crc32` since zlib hardcodes the ISO polynomial. The options:

**Option A: `crc32c` package** (recommended)
```python
# Instead of: import zlib
import crc32c

# Instead of: zlib.crc32(data) & 0xFFFFFFFF
crc32c.crc32c(data)
```
The `crc32c` package auto-detects and uses hardware SSE 4.2/ARM instructions. No mask needed — it always returns unsigned.

**Option B: `crcmod`**
```python
import crcmod
crc32c_fn = crcmod.predefined.mkCrcFun('crc-32c')
checksum = crc32c_fn(data)
```
More general but slightly more ceremony. Also supports hardware acceleration depending on build.

The change touches 5 files (`write-ahead-log/wal.py`, `b-tree-storage-engine/btree.py`, `log-structured-hash-table/bitcask.py`, and two test files) across 13 call sites. It's a mechanical replacement, but it's a **wire format change** — any existing data files would fail CRC verification after switching. A real migration would need a version flag in the file header to know which polynomial was used, similar to how PostgreSQL added CRC-32C in version 9.5 with a WAL format version bump.

## Notable gap

The `log-structured-merge-tree/lsm.py` module has **no checksumming at all** — neither its WAL nor its SSTables compute CRCs. Compare its `WAL.append` (just writes raw key/value with length prefixes) against the checksummed WAL in `write-ahead-log/wal.py:28-34`. A corrupt byte in an LSM SSTable would be silently returned as garbage.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_checksum` — Isolated CRC wrapper; good place to experiment with swapping polynomials and benchmarking
- [file] `log-structured-merge-tree/lsm.py` — No integrity checking at all; worth understanding what RocksDB does differently with block checksums in its SSTable format
- [general] `sse42-crc32c-benchmarking` — Measure the actual throughput difference between `zlib.crc32` and hardware-accelerated `crc32c` on your machine with realistic page sizes
- [function] `write-ahead-log/wal.py:_read_record` — CRC verification on the read path; consider what happens if you need to support both old (ISO) and new (Castagnoli) WAL files during migration
- [general] `xxhash-vs-crc32c` — Some modern systems (e.g., newer RocksDB options) use xxHash instead of CRC-32C for non-cryptographic checksums; explore the tradeoffs

## Beliefs

- `zlib-crc32-is-iso3309` — All 13 CRC call sites use `zlib.crc32`, which implements the ISO 3309 polynomial (0xEDB88320), not the Castagnoli polynomial used by RocksDB and PostgreSQL
- `lsm-has-no-checksumming` — The LSM tree module (`lsm.py`) performs no CRC or checksum verification on WAL entries or SSTable records, unlike the standalone WAL and B-tree modules
- `crc-mask-is-redundant-python3` — The `& 0xFFFFFFFF` mask applied after `zlib.crc32` in all modules is a Python 2 compatibility pattern; Python 3's `zlib.crc32` already returns an unsigned 32-bit integer
- `polynomial-switch-breaks-wire-format` — Changing from ISO 3309 to Castagnoli CRC-32C is a backward-incompatible wire format change; existing data files would fail verification without a format version mechanism

