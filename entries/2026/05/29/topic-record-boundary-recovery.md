# Topic: Compare this "stop at first error" approach with alternatives like length-prefixed scanning with magic bytes or block-aligned records (as in LevelDB's log format)

**Date:** 2026-05-29
**Time:** 07:31

# "Stop at First Error" vs. Alternative Recovery Strategies

## How This Codebase Handles Corruption

Every log reader in this repo follows the same pattern: **scan forward sequentially, stop the moment anything looks wrong**. There are no attempts to skip past corruption and recover later records.

The WAL implementation in `write-ahead-log/wal.py:85-95` shows this most explicitly:

```python
# _recover_seq_num
while True:
    try:
        rec = _read_record(f)
        if rec is None:       # EOF or short read
            break
        max_seq = max(max_seq, rec.seq_num)
    except ValueError:        # CRC mismatch
        break                 # ← abandon the rest of this file
```

The `_read_record` function (`wal.py:38-58`) uses length-prefixed framing — it reads a 4-byte record length, then reads exactly that many bytes — but if the CRC fails, it raises `ValueError`, and every caller responds by breaking out of the scan loop entirely.

The Bitcask implementation in `log-structured-hash-table/bitcask.py:89-105` does the same thing without the exception:

```python
# _scan_segment
if crc != expected_crc:
    break  # corrupted record at end, stop scanning
```

The LSM tree's WAL (`log-structured-merge-tree/lsm.py:31-52`) is even more bare — no CRC at all, just length-prefixed fields with short-read guards. A single corrupted length field would cause it to read garbage offsets, and there's no way to recover.

## What This Approach Gives Up

The "stop at first error" strategy is **correct for tail corruption** — the common case where a crash interrupts a write in progress, leaving a partial record at the end of the file. Everything before it is intact; everything after it doesn't exist. The test in `test_wal.py:44-58` validates exactly this: corrupt the last 5 bytes, recover, get back only the first record.

But this strategy **silently discards valid records** that follow a mid-file corruption. If a disk sector goes bad in the middle of the file, or a cosmic ray flips a bit in an earlier record, every record after the corruption is lost — even though they're perfectly valid and fully written.

## Alternative 1: Block-Aligned Records (LevelDB's Approach)

LevelDB's log format divides the file into fixed-size 32KB blocks. Each record within a block carries its own header: `checksum (4B) | length (2B) | type (1B)`. The type field handles records that span block boundaries (`FIRST`, `MIDDLE`, `LAST`, `FULL`).

The key insight: **if corruption is found, skip to the next 32KB block boundary and resume scanning**. You lose at most ~32KB of data instead of the entire tail.

Compared to this codebase's WAL format (`wal.py:28-34`):
- The WAL uses variable-length records with a leading `record_length` field. If that 4-byte length is corrupted, the reader jumps to a garbage offset and can never resync.
- LevelDB's block alignment means you always know where the next potential record boundary is — it's a function of arithmetic, not of the data.

The tradeoff: block alignment wastes space at block boundaries (padding bytes when a record doesn't fit the remaining space), and records that span blocks need the FIRST/MIDDLE/LAST machinery. The WAL here packs records tightly with zero waste.

## Alternative 2: Magic Bytes for Resynchronization

Some formats prepend a magic number (e.g., `0xCAFEBABE`) to every record. On corruption, you scan forward byte-by-byte looking for the next magic sequence, then try to parse from there.

None of the implementations here use magic bytes. The Bitcask format (`log-structured-hash-table/bitcask.py:12-13`) goes straight into `crc32 | key_size | value_size` — no sentinel. The WAL format (`wal.py:33`) starts with `record_length | crc | seq_num | op_type | key_len` — also no sentinel.

Magic bytes would let these readers resync after mid-file corruption, but with costs:
- **False positives**: the magic sequence can appear in user data, requiring validation after every candidate match
- **Scan overhead**: byte-by-byte scanning is slow compared to structured reads
- **Complexity**: the reader needs a "scanning" mode separate from its normal "parsing" mode

## Why "Stop at First Error" Is Reasonable Here

For these reference implementations, the design is pragmatic:

1. **WAL files are short-lived**. They're truncated after checkpoint (`wal.py:176-200`), and the LSM WAL is truncated after flush (`lsm.py:56-60`). Mid-file corruption in a file that exists for seconds to minutes is unlikely.

2. **Segment rotation limits blast radius**. Both Bitcask stores rotate segments at a size threshold (`bitcask.py:118`, `hash-index-storage/bitcask.py:101`). Losing the tail of one segment doesn't affect other segments.

3. **Simplicity matters for correctness**. Block-aligned formats add ~100 lines of boundary-management code. Each line is a place for bugs. For a teaching codebase demonstrating DDIA concepts, the simpler approach is easier to audit.

The real cost shows up in production-grade systems where WAL files can grow large before checkpointing, or where storage media failures cause mid-file corruption. That's where LevelDB's block format earns its complexity.

## Topics to Explore

- [general] `leveldb-log-format` — Read the LevelDB `log_format.md` and `log_reader.cc` to see how block-aligned recovery handles the FIRST/MIDDLE/LAST record spanning and the `kBadRecord` skip logic
- [function] `write-ahead-log/wal.py:_encode_record` — Trace how the record_length prefix creates a chain of trust where one corrupted length field breaks all subsequent parsing
- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — Compare the CRC-based stop-at-error here with the LSM WAL's no-CRC approach in `lsm.py:replay` and consider what each loses on corruption
- [file] `write-ahead-log/test_wal.py` — The corruption test only covers tail corruption (last 5 bytes); try inserting corruption mid-file to observe how many valid trailing records are silently dropped
- [general] `sstable-footer-integrity` — The SSTable format in `lsm.py` stores its sparse index as a footer; explore what happens if the footer's 12-byte trailer is corrupted vs. a mid-file data entry

## Beliefs

- `wal-stop-at-first-error` — All WAL and segment readers in this codebase break out of the scan loop on the first CRC failure or short read, discarding all subsequent records regardless of their validity
- `wal-no-resync-capability` — No implementation uses magic bytes, block alignment, or any other mechanism that would allow resuming parsing after a corrupted record in the middle of a file
- `lsm-wal-no-crc` — The LSM tree's WAL (`lsm.py:14-53`) uses only length-prefixed framing with no checksum, making it unable to distinguish corruption from valid data unless a length field causes an out-of-bounds read
- `segment-rotation-limits-corruption-blast-radius` — Both Bitcask implementations rotate to new segment files at a configurable size threshold, bounding the maximum data loss from stop-at-first-error to the records trailing the corruption within a single segment

