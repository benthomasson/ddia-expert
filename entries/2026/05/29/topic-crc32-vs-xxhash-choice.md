# Topic: All three use `zlib.crc32`; whether this is adequate for the file sizes involved or whether a faster/stronger hash would change the design

**Date:** 2026-05-29
**Time:** 08:14

# CRC32 Adequacy Across the Storage Engines

## What CRC32 Is Doing Here

All three implementations use `zlib.crc32` for the same purpose: **per-record integrity checks** — detecting corruption in individual records after they've been written to disk. This is *not* content addressing, deduplication, or cryptographic verification. The question is whether CRC32 is the right tool for that job at the data sizes these engines handle.

## The File Sizes Involved

The defaults tell the story:

- **WAL** (`write-ahead-log/wal.py:65`): `max_file_size = 10 * 1024 * 1024` (10 MB per segment)
- **Bitcask** (`log-structured-hash-table/bitcask.py:10`): `HEADER_FMT = "!III"` — each record is a header (12 bytes) plus a key-value payload. Segment files are similarly bounded.
- **B-tree** (`b-tree-storage-engine/btree.py:176`): CRC covers individual pages/nodes, typically 4–16 KB each.

But the CRC doesn't cover the *file*. It covers individual *records* or *pages*. In the WAL, `_encode_record` (line 29–33) checksums `op_type_byte + key + value` — a single operation's payload, typically bytes to low kilobytes. In Bitcask, the CRC covers `payload` (key + value bytes). In the B-tree, it covers a single page's data buffer.

**This is the critical point: CRC32 isn't protecting 10 MB files. It's protecting individual records that are typically a few hundred bytes to a few KB.**

## Is CRC32 Adequate?

**Yes, for this use case.** Here's why:

### Collision probability is negligible at per-record granularity

CRC32 has a 32-bit output space (4.3 billion values). The probability of a random corruption going undetected is ~1 in 4.3 billion per record. For these implementations — educational reference code handling at most thousands of records — the expected number of undetected corruptions is vanishingly small. You'd need to be checking billions of records before birthday-bound collisions become a concern, and even then CRC32's polynomial properties mean it catches all single-bit errors, all double-bit errors, and all burst errors up to 32 bits — which covers the vast majority of real disk/memory corruption patterns.

### Performance is a non-issue

`zlib.crc32` in CPython delegates to a C implementation and is hardware-accelerated on modern x86 (via SSE 4.2's CRC32C instruction, though `zlib.crc32` uses the CRC-32 polynomial, not CRC-32C). It processes data at multiple GB/s. For records of a few hundred bytes, the CRC computation is noise compared to the `fsync` calls that dominate WAL latency (`wal.py:126–128`) and the file I/O in Bitcask and B-tree reads.

### What a stronger hash would *not* change

Switching to SHA-256 or xxHash wouldn't change the design at all:

1. **The record format stays the same** — you'd just widen the checksum field from 4 bytes to 32 bytes (SHA-256) or keep it at 4–8 bytes (xxHash). The encode/decode logic in `_encode_record` / `_read_record` is structurally identical.
2. **The error handling stays the same** — all three implementations raise or skip on mismatch (`wal.py:55–56`: `raise ValueError(f"CRC mismatch at seq {seq_num}")`; `bitcask.py:99–100` checks `expected_crc`). A stronger hash still produces a boolean "matches or doesn't."
3. **No architectural decisions depend on hash strength** — nobody uses the CRC for routing, indexing, or dedup. It's purely a corruption detector.

### What *would* matter in production

In a production system at scale, two upgrades are common but neither changes the architecture:

- **CRC-32C** (Castagnoli polynomial) instead of CRC-32 (ISO 3309). CRC-32C has better error-detection properties for certain burst patterns and is directly accelerated by Intel's `crc32` instruction. PostgreSQL, RocksDB, and LevelDB all use CRC-32C. But Python's `zlib.crc32` uses the ISO polynomial — switching to CRC-32C would require a different library (e.g., `crcmod` or `google-crc32c`).
- **xxHash (xxh64/xxh128)** for speed on large pages. xxHash is ~3× faster than CRC32 on large inputs without hardware acceleration. But at the per-record sizes here, the difference is immeasurable.

## The Interesting Contrast

Notice that the codebase *does* use `hashlib` (SHA-256) in other modules — `merkle-tree/merkle_tree.py`, `bloom-filter/bloom_filter.py`, `consistent-hashing/consistent_hashing.py`, and `byzantine-fault-tolerance/pbft.py`. Those modules need either **cryptographic security** (BFT), **uniform distribution** (consistent hashing, bloom filters), or **content addressing** (Merkle trees). The storage engines don't need any of those properties — they need fast corruption detection, which is exactly what CRC32 provides.

Also notable: the SSTable implementation (`sstable-and-compaction/sstable.py`) and the LSM tree's SSTable (`log-structured-merge-tree/lsm.py`) use **no checksums at all**. They rely on the magic bytes (`MAGIC = b"SSTB"`) for format validation but have no per-entry integrity check. If you were going to change something about hashing in this codebase, adding CRC32 to the SSTable format would be a more impactful improvement than upgrading the existing CRC32 users to a stronger hash.

## Topics to Explore

- [file] `sstable-and-compaction/sstable.py` — No per-entry checksums despite being a persistent format; compare with the WAL and Bitcask approaches to understand when checksums are considered essential vs. optional
- [function] `write-ahead-log/wal.py:_encode_record` — Study what exactly is included in the CRC input (op_type + key + value, but NOT seq_num or record_length) and consider whether excluding header fields from the checksum is a vulnerability
- [general] `crc32-vs-crc32c` — Why production systems (RocksDB, PostgreSQL) prefer the Castagnoli polynomial over ISO 3309, and what Python library changes that would require
- [file] `merkle-tree/merkle_tree.py` — Contrast SHA-256 usage for content addressing with CRC32 for corruption detection to understand when each hash family is appropriate
- [function] `log-structured-merge-tree/lsm.py:SSTable.write` — Compare this checksumless SSTable format with the checksummed one in `sstable-and-compaction/sstable.py` to see two different design choices for the same data structure

## Beliefs

- `crc32-covers-records-not-files` — CRC32 is computed per individual record/page (hundreds of bytes), not per file; file sizes (10 MB default) are irrelevant to collision probability
- `sstable-lacks-checksums` — The SSTable implementations in `sstable-and-compaction/sstable.py` and `log-structured-merge-tree/lsm.py` have no per-entry integrity checks, unlike WAL and Bitcask
- `crc32-input-excludes-metadata` — In `wal.py:30–31`, the CRC input is `op_type_byte + key + value` but excludes `seq_num` and `record_length`, meaning corruption in those header fields would not be detected by the checksum
- `hashlib-reserved-for-crypto-distribution` — The codebase consistently uses `zlib.crc32` for corruption detection and `hashlib` (SHA-256) for content addressing, uniform distribution, and cryptographic integrity — the two never cross purposes

