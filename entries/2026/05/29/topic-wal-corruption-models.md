# Topic: The difference between tail corruption (interrupted append) vs mid-file corruption (bit rot) and how production WALs like SQLite/RocksDB handle each case

**Date:** 2026-05-29
**Time:** 08:17

# Tail Corruption vs Mid-File Corruption in WALs

## The Two Failure Modes

These are fundamentally different problems with different causes, detection mechanisms, and recovery strategies.

**Tail corruption** happens when a process crashes mid-write. The last record in the file is incomplete — maybe only half the bytes made it to disk before power died. The file is valid up to the point of interruption, then garbage. This is the *expected* failure mode for any append-only log.

**Mid-file corruption** (bit rot) happens when bits flip in storage — hardware faults, cosmic rays, firmware bugs, silent disk errors. A record that was once valid now has wrong bytes, but the records before *and after* it may be perfectly fine. This is rarer but more insidious because it can go undetected for a long time.

## How This Codebase Handles Each

### Tail Corruption (Partial Reads)

Look at `_read_record` in `write-ahead-log/wal.py:38-58`. The function handles tail corruption by checking whether it got enough bytes:

```python
length_data = f.read(4)
if len(length_data) < 4:
    return None          # partial header → treat as EOF
record_data = f.read(record_length)
if len(record_data) < record_length:
    return None          # partial body → treat as EOF
```

Returning `None` (rather than raising) is the key design choice. The callers — `_recover_seq_num` at line 89 and `iterate` at line 233 — treat `None` as "end of usable data" and stop reading. This is correct: if the tail is torn, everything before it was fsync'd and valid; everything after is junk from an interrupted write.

The LSM WAL in `log-structured-merge-tree/lsm.py:36-53` uses the identical pattern — boundary checks on every `read()` call, breaking silently on short reads:

```python
if pos + 4 > len(data):
    break
```

### Mid-File Corruption (CRC Mismatch)

Now look at the CRC check in `write-ahead-log/wal.py:53-56`:

```python
crc_data = struct.pack("B", op_type_byte) + key + value
expected_crc = zlib.crc32(crc_data) & 0xFFFFFFFF
if crc != expected_crc:
    raise ValueError(f"CRC mismatch at seq {seq_num}")
```

This **raises** rather than returning `None`. That's the distinction: a short read is benign (tail corruption), but a full record with a bad checksum is evidence of actual data corruption.

However, the callers don't differentiate well. Both `_recover_seq_num` (line 93) and `replay` (line 215) catch the `ValueError` and just `break`:

```python
except ValueError:
    break
```

This means **mid-file corruption is treated identically to tail corruption** — everything after the bad record is silently discarded, even if later records are perfectly valid. The test at `test_wal.py:44-57` confirms this: it corrupts the last 5 bytes of the file, and replay returns only the first record, discarding the second.

## How Production Systems Do Better

### SQLite WAL

SQLite's WAL uses per-frame checksums (similar to the per-record CRC here) but adds two critical mechanisms this implementation lacks:

1. **Cumulative checksums**: Each frame's checksum chains from the previous frame's checksum value. This means a valid frame N proves that frames 0 through N are all intact — you don't need to re-verify earlier frames, and a corrupt frame in the middle can't be masked by valid-looking frames after it.

2. **Salt values in the WAL header**: The header contains two random salt values that change on each WAL reset. Every frame's checksum includes the salt. This prevents stale frames from a previous WAL generation from being mistaken for valid data in the current generation — a tail corruption scenario that simple CRC alone can't catch.

For tail corruption specifically, SQLite simply stops replaying at the first frame with a bad checksum, much like this implementation. But because checksums are cumulative, a mid-file corruption *also* invalidates all subsequent frames, making the two cases collapse into one.

### RocksDB WAL

RocksDB takes a different approach with its block-based WAL format (inherited from LevelDB):

1. **Fixed-size blocks (32KB)**: The WAL is divided into fixed blocks. Each record fragment within a block carries its own CRC, a type tag (FULL, FIRST, MIDDLE, LAST), and a length. This means a single logical record can span multiple blocks.

2. **Per-block recovery**: Because blocks are fixed-size and self-contained, mid-file corruption in block N doesn't prevent reading block N+1. RocksDB's recovery modes make this configurable:
   - `kTolerateCorruptedTailRecords` — skip only the torn tail (the default, handles the common crash case)
   - `kAbsoluteConsistency` — any corruption is fatal
   - `kPointInTimeRecovery` — stop at the first corruption, like this codebase
   - `kSkipAnyCorruptedRecords` — skip corrupt records but keep reading valid ones after them

The key insight: RocksDB can **skip past** mid-file corruption because fixed-size blocks let you find the next valid record boundary. This codebase's variable-length records (line 32: `record_length` prefix) make that impossible — if a record's length field is corrupted, you can't find where the next record starts.

## What's Missing From This Implementation

The Bitcask implementation in `hash-index-storage/bitcask.py` has **no checksums at all** (lines 84-92 show the record format is just header + key + value with no CRC). A single bit flip anywhere is completely undetectable. The `assert read_key == key` at line 177 is the only sanity check, and it only catches corruption in the key field.

The B-tree implementation (`b-tree-storage-engine/btree.py:120-176`) does better — it has per-page CRC32 checksums in its WAL entries, and crucially, it validates the checksum during *replay* (line 164: `if self._checksum(page_data) == checksum`), silently skipping corrupt entries rather than crashing. This skip-and-continue approach is closer to RocksDB's `kSkipAnyCorruptedRecords` mode.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_record` — Trace how `None` vs `ValueError` propagates through all callers to understand the recovery semantics
- [function] `b-tree-storage-engine/btree.py:_checksum` — Compare the B-tree's skip-corrupt-entries approach to the WAL's stop-at-first-corruption approach and consider which is safer
- [general] `block-based-wal-format` — Study LevelDB/RocksDB's 32KB block format to understand how fixed-size blocks enable recovery past mid-file corruption
- [general] `cumulative-checksums` — Explore how SQLite's chained frame checksums prevent a class of corruption that per-record CRCs miss
- [file] `hash-index-storage/bitcask.py` — Examine why the Bitcask implementation lacks checksums entirely and what failure modes that creates during compaction

## Beliefs

- `wal-tail-corruption-silent-truncation` — `_read_record` returns `None` on short reads (lines 40-43), causing all callers to silently discard partial tail records without error
- `wal-mid-corruption-stops-replay` — A CRC mismatch raises `ValueError` (line 56), and both `replay` and `_recover_seq_num` catch it with `break`, discarding all records after the corruption point — including valid ones
- `wal-crc-excludes-length-and-seqnum` — The CRC at line 30-31 covers only `op_type + key + value`, not the record length or sequence number, so corruption in those header fields produces unpredictable behavior rather than a clean CRC failure
- `bitcask-has-no-integrity-checks` — The Bitcask store (`bitcask.py`) writes and reads records with no checksums; the only corruption guard is a key-match assert at line 177
- `lsm-wal-has-no-checksums` — The LSM tree's WAL (`lsm.py:13-63`) uses length-prefixed records with no CRC, relying solely on short-read detection for crash recovery

