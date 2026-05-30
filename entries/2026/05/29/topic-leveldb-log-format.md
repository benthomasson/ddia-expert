# Topic: Read the LevelDB `log_format.md` and `log_reader.cc` to see how block-aligned recovery handles the FIRST/MIDDLE/LAST record spanning and the `kBadRecord` skip logic

**Date:** 2026-05-29
**Time:** 10:05

# LevelDB Block-Aligned WAL Recovery vs. This Codebase

## The Gap: What's Missing

The observations are **insufficient** to directly explain LevelDB's `log_format.md` and `log_reader.cc` — those are files in the LevelDB C++ codebase (Google's `leveldb/doc/log_format.md` and `leveldb/db/log_reader.cc`), not files in this repository. Neither WAL implementation here (`write-ahead-log/wal.py` or `log-structured-merge-tree/lsm.py`) uses LevelDB's block-aligned record format. Let me explain the LevelDB design and how this codebase diverges from it.

## LevelDB's Block-Aligned Record Format

LevelDB divides the WAL into **fixed 32KB blocks** (`kBlockSize = 32768`). Every record written to the log has a 7-byte header (4-byte CRC, 2-byte length, 1-byte type). When a logical record doesn't fit in the remaining space of the current block, it gets **fragmented** across blocks using record types:

| Type | Meaning |
|------|---------|
| `FULL` | Entire record fits in one block fragment |
| `FIRST` | First fragment of a multi-block record |
| `MIDDLE` | Interior fragment (neither first nor last) |
| `LAST` | Final fragment of a multi-block record |

The reader reassembles by accumulating fragments: when it sees `FIRST`, it starts a buffer; `MIDDLE` appends to it; `LAST` appends and delivers the complete record. A `FULL` record delivers immediately.

### The `kBadRecord` Skip Logic

In `log_reader.cc`, `ReadPhysicalRecord()` reads one physical record (one fragment) from the current block. It returns `kBadRecord` when:

1. **CRC mismatch** — the checksum doesn't match the payload
2. **Length overflows the block** — the header claims more bytes than remain in the current 32KB block
3. **Zero-length record in block trailer** — the last 6 bytes of a block can't hold a header + any payload, so they're filled with zeros (a "trailer"); the reader skips them

When `ReadRecord()` (the logical-record reader) gets `kBadRecord` from `ReadPhysicalRecord()`, its behavior depends on context:

- **If currently assembling a fragmented record** (saw `FIRST` or `MIDDLE`): it **discards the entire in-progress record** and reports the corruption, then resets its state
- **If not assembling**: it simply skips and tries the next physical record
- **`initial_offset` seeking**: when recovering from a specific point (not the start of the file), the reader skips forward to the next block boundary and ignores `kBadRecord` and unexpected types until it finds a valid `FIRST` or `FULL` — this is how it resynchronizes after corruption

The block alignment is the key insight: **you can always find the next valid record by jumping to the next 32KB boundary**. Corruption in one block doesn't cascade — the reader just skips to the next block and looks for a `FIRST` or `FULL` type.

## How This Codebase Differs

### `write-ahead-log/wal.py` — No Block Alignment

The `_read_record()` function at line 37 reads a 4-byte length prefix, then reads that many bytes. There are no blocks, no fragment types, no `FIRST`/`MIDDLE`/`LAST`. Records are variable-length and packed contiguously.

The consequence shows at `wal.py:92` and `wal.py:238` — when recovery hits a CRC error or a short read, it **stops entirely** (`break`). There's no way to skip past corruption because without block boundaries, there's no reliable resynchronization point. A single corrupted byte in the middle of the file means every record after it is lost.

### `log-structured-merge-tree/lsm.py` — Even Simpler

The `WAL` class (line 13) uses bare `struct.pack(">I", len(k))` length-prefixed encoding with no CRC at all. The `replay()` method (line 29) handles truncation (short reads) but has **zero corruption detection**. If a byte is flipped in the middle of the file, you get silently wrong data rather than an error.

### What LevelDB's Design Buys You

| Property | LevelDB | `wal.py` | `lsm.py` WAL |
|----------|---------|----------|---------------|
| Corruption detection | CRC-32 per fragment | CRC-32 per record | None |
| Recovery after corruption | Skip to next 32KB block | Stop at corruption | Stop at truncation |
| Large record support | Fragment across blocks | Single write (no limit) | Single write |
| Seek to arbitrary point | Align to block boundary | Scan from start | Scan from start |

The block-aligned design is a classic space-time tradeoff: you waste up to 6 bytes per block on trailers, but gain the ability to recover the tail of a WAL file even when the middle is damaged.

## Topics to Explore

- [general] `leveldb-block-alignment-tradeoffs` — Investigate why LevelDB chose 32KB blocks specifically — it's a tradeoff between wasted trailer space, mmap granularity, and the probability of losing records adjacent to corruption
- [function] `write-ahead-log/wal.py:_read_record` — Consider adding block-aligned framing to this WAL so recovery can skip past mid-file corruption instead of stopping
- [file] `log-structured-merge-tree/lsm.py` — The WAL class here has no checksums at all; a single bit flip produces silent data corruption during replay
- [general] `wal-record-batching-atomicity` — Compare how `wal.py:append_batch` (line 152) achieves atomic batch writes vs. LevelDB's WriteBatch encoding inside a single logical record that may span multiple physical fragments
- [function] `write-ahead-log/wal.py:_recover_seq_num` — This scans all WAL files sequentially from byte 0; with block-aligned format, you could binary-search for the last valid block instead

## Beliefs

- `wal-py-stops-at-first-corruption` — `_read_record()` in `wal.py:37` breaks on CRC mismatch or short read, discarding all subsequent records rather than attempting resynchronization
- `lsm-wal-has-no-checksums` — The `WAL` class in `lsm.py:13` uses bare length-prefixed encoding with no CRC; bit flips produce silent data corruption during replay
- `no-block-aligned-wal-in-codebase` — Neither WAL implementation uses fixed-size block framing; both use variable-length contiguous records, preventing skip-past-corruption recovery
- `wal-py-crc-covers-optype-key-value` — The CRC in `wal.py:31` is computed over `op_type_byte + key + value` but excludes the sequence number and length fields, meaning corruption in those header fields produces a short/wrong read rather than a CRC error

