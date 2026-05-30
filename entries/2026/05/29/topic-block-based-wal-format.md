# Topic: Study LevelDB/RocksDB's 32KB block format to understand how fixed-size blocks enable recovery past mid-file corruption

**Date:** 2026-05-29
**Time:** 12:07

# Fixed-Size Blocks and Corruption Recovery

## What the codebase does — and the gap it reveals

This codebase implements two SSTable/WAL designs, and **neither uses fixed-size blocks**. That's exactly the point: understanding what goes wrong without them is the best way to understand why LevelDB/RocksDB chose the 32KB block format.

### The current record format: variable-length, no block boundaries

In `sstable-and-compaction/sstable.py:60-75`, entries are written as variable-length records back-to-back:

```
[key_len:2][key:N][timestamp:8][tombstone_marker | value_len:4 + value:M]
```

The `block_size` parameter at line 46 is misleading — it controls the **sparse index interval** (how many entries between index bookmarks), not a physical block size in bytes. At line 62, `if self._count % self._block_size == 0` just decides when to record an index entry. Records still flow contiguously with no alignment or padding to fixed boundaries.

The LSM tree's SSTable in `log-structured-merge-tree/lsm.py:80-98` has the same shape: variable-length key-value pairs, sparse index in the footer, no fixed block structure.

### Why this breaks under mid-file corruption

If a byte gets flipped or a sector goes bad at offset 5000 in either format, the reader (`_read_entry` at `sstable.py:141-163` or `lsm.py:183-196`) will read a garbage `key_len`, seek forward by that garbage count, and never resynchronize. **Every record after the corruption point is lost**, even though those bytes are perfectly intact.

The WAL at `write-ahead-log/wal.py:37-58` is slightly better — it has CRC32 checksums per record (line 30-31) and a length prefix (line 33), so `_read_record` detects corruption via CRC mismatch at line 55. But it still **stops at corruption** (line 215: "stops at corruption"). It cannot skip ahead to find the next valid record because records are variable-length — there's no predictable offset where the next record might start.

### How LevelDB/RocksDB's 32KB block format solves this

LevelDB's log format (which RocksDB inherits) divides files into **fixed 32KB (32768-byte) blocks**. Each block contains a sequence of records, each with:

```
[checksum:4][length:2][type:1][data:N]
```

The `type` field encodes whether a record is `FULL`, `FIRST`, `MIDDLE`, or `LAST` — allowing a single logical record to span multiple blocks.

The critical recovery property: **if you hit corruption, you skip to the next 32KB boundary and resume reading**. Since every block starts at a multiple of 32768 bytes, the reader can always compute the next boundary with `(current_offset / 32768 + 1) * 32768`. You lose only the records in the corrupted block, not everything after it.

This is a property that **no variable-length format can provide** without fixed-size blocks, because without predictable boundaries, there's no address to seek to after corruption.

### What would need to change in this codebase

To add LevelDB-style corruption recovery, the SSTable writer would need to:

1. Pad each block to exactly 32KB (wasting the trailing bytes)
2. Add a checksum per block, not per record
3. Fragment records that span block boundaries into typed chunks (FIRST/MIDDLE/LAST)
4. On read, validate each block's checksum before parsing, and skip entire blocks that fail

The current sparse index (line 62) could remain — but it would index block-aligned offsets instead of arbitrary file positions, making block skipping and index lookups complementary.

### The tradeoff

Fixed-size blocks waste space at block boundaries (internal fragmentation from padding). The 32KB size is a compromise: small enough that a single corrupted block doesn't lose much data, large enough that the padding overhead and per-block checksum cost are amortized. LevelDB's original design doc cites ~0.03% overhead for the block headers at 32KB.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_record` — Shows CRC-based corruption detection that stops at first error; contrast with block-skip recovery
- [function] `sstable-and-compaction/sstable.py:_read_entry` — Variable-length deserialization that cannot resynchronize after corruption
- [general] `leveldb-log-format` — Study LevelDB's `log_format.md` and `log_reader.cc` to see the actual FIRST/MIDDLE/LAST fragment protocol
- [function] `log-structured-merge-tree/lsm.py:SSTable.write` — Compare the footer-based sparse index with what a block-aligned index would look like
- [file] `write-ahead-log/test_wal.py` — The `test_corruption` test (line 44) demonstrates stop-at-corruption behavior; extending it to test skip-past-corruption would validate a block-based design

## Beliefs

- `sstable-block-size-is-entry-count` — SSTableWriter's `block_size` parameter controls sparse index frequency (entries per index point), not a physical byte-aligned block size (`sstable.py:62`)
- `wal-corruption-stops-replay` — WAL replay halts at the first CRC mismatch and cannot recover records beyond the corruption point (`wal.py:215`, `wal.py:233`)
- `no-fixed-block-boundaries-in-sstable` — Both SSTable implementations write variable-length records contiguously with no padding or alignment to fixed byte boundaries, making mid-file resynchronization impossible
- `wal-has-per-record-checksums` — The WAL uses CRC32 per record (`wal.py:30-31`) while the SSTable data region has no checksums at all, making silent corruption in SSTables undetectable

