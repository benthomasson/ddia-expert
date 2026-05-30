# Topic: Study how LevelDB's FIRST/MIDDLE/LAST fragment types handle records larger than 32 KB and what happens when a crash lands between fragments

**Date:** 2026-05-29
**Time:** 11:01

# LevelDB Fragment Types: What This Implementation Does (and Doesn't Do)

## The Short Answer: This WAL Has No Fragment Types

The observations are clear — every grep for `FIRST`, `MIDDLE`, `LAST`, `FULL`, `BLOCK_SIZE`, `32768`, and `fragment` returned **zero matches**. The WAL in `write-ahead-log/wal.py` uses a simpler design that does not implement LevelDB's block-based log format.

## What This Implementation Does Instead

The WAL at `write-ahead-log/wal.py:28-55` writes each record as a single contiguous blob:

```
[4 bytes: record_length][4 bytes: CRC][8 bytes: seq_num][1 byte: op_type][4 bytes: key_len][key][4 bytes: val_len][value]
```

There is no concept of fixed-size blocks. A record of any size — 100 bytes or 100 KB — is written as one unit. The `_read_record` function at line 39 reads the 4-byte length prefix, then reads that many bytes as a single record. The `test_large_values` test (line 116) confirms this works for a 100,000-byte value with no fragmentation at all.

This means crash recovery is straightforward: `_read_record` returns `None` on a short read (lines 42-43, 46-47), and raises `ValueError` on a CRC mismatch (line 54). A torn write at the tail is simply discarded.

## What LevelDB Does Differently (and Why It Matters)

LevelDB's log format divides the log file into fixed **32 KB blocks** (32,768 bytes). Each block contains a sequence of fragments, where each fragment has a 7-byte header:

```
[4 bytes: CRC][2 bytes: length][1 byte: type]
```

The `type` byte is one of four values:

| Type | Meaning |
|------|---------|
| **FULL** | The entire record fits within this block |
| **FIRST** | Start of a record that spans multiple blocks |
| **MIDDLE** | Continuation of a multi-block record (can repeat) |
| **LAST** | Final fragment of a multi-block record |

A 70 KB record would be split into: FIRST (filling the remainder of the current block) → MIDDLE (filling an entire 32 KB block) → LAST (the remaining bytes in the next block).

### Why Blocks?

Fixed-size blocks enable **random-access seeking**. To skip to byte offset 1,000,000, you can jump to block `floor(1000000 / 32768)` and start reading fragment headers from the block boundary. You don't need to scan from the beginning of the file. This is impossible with the variable-length record format in `wal.py`, where you must sequentially read every length prefix to find record boundaries.

### What Happens When a Crash Lands Between Fragments

This is the critical design question. If a record spans three blocks (FIRST → MIDDLE → LAST) and the process crashes after writing FIRST but before writing MIDDLE:

1. **On recovery**, the reader encounters a FIRST fragment followed by either EOF or a non-MIDDLE/LAST fragment type.
2. **LevelDB discards the incomplete record** — the FIRST fragment is silently dropped because no LAST was seen to complete it.
3. **Each fragment has its own CRC**, so a partially-written fragment within a block is also detected and dropped.
4. **Subsequent records are not lost** — because block boundaries are discoverable by alignment, the reader can skip to the next 32 KB boundary and resume scanning even if the current block is corrupted.

This is a significant advantage over the approach in `wal.py`. In the current implementation, if `_read_record` encounters corruption at line 54, recovery **stops entirely** — the `_recover_seq_num` loop at line 89 breaks on `ValueError`, and any valid records after the corrupt one are lost.

## What's Missing From These Observations

The codebase does not contain a LevelDB-style log implementation. To study the FIRST/MIDDLE/LAST design in code, you would need:

- A block-oriented writer that splits records at 32 KB boundaries and assigns fragment types
- A reader that reassembles fragments across blocks and handles incomplete fragment sequences
- Recovery logic that can seek to block boundaries and skip corrupted blocks without losing subsequent data

The `test_large_values` test at line 116 proves the current design handles large records, but through a fundamentally different mechanism (no block boundaries, no fragmentation).

## Topics to Explore

- [general] `leveldb-log-format-spec` — Read LevelDB's `doc/log_format.md` in the upstream LevelDB repo to see the exact block/fragment wire format and the state machine for reassembly
- [function] `write-ahead-log/wal.py:_read_record` — Consider how this function would need to change to support block-aligned reads and fragment reassembly instead of length-prefixed whole records
- [general] `block-aligned-corruption-recovery` — Study how fixed-size blocks allow skipping past corruption to the next block boundary, compared to the current approach where corruption halts recovery
- [function] `write-ahead-log/wal.py:append_batch` — Compare this atomicity mechanism (buffer all + single write + fsync) with how LevelDB achieves batch atomicity across fragments that may span block boundaries
- [file] `write-ahead-log/test_wal.py` — The corruption test at line 48 only corrupts the tail; a block-based format would allow testing mid-file corruption with continued recovery of later records

## Beliefs

- `wal-no-block-fragmentation` — The WAL implementation uses variable-length records with no fixed block boundaries or fragment types; a record of any size is written and read as a single contiguous unit
- `wal-corruption-halts-recovery` — When `_read_record` hits a CRC mismatch or short read, recovery stops completely and all subsequent valid records in that file are lost (wal.py lines 85-91)
- `wal-large-records-no-limit` — Records are not bounded by block size; `test_large_values` confirms a 100 KB value is stored and replayed as a single record with no fragmentation
- `leveldb-fragment-crash-safety` — In LevelDB's design, a crash between fragments of a multi-block record causes only that record to be lost; block-aligned seeking allows recovery of all records after the gap

