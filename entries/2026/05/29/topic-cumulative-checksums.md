# Topic: Explore how SQLite's chained frame checksums prevent a class of corruption that per-record CRCs miss

**Date:** 2026-05-29
**Time:** 12:10

# Chained Frame Checksums vs. Per-Record CRCs

## The core problem: per-record CRCs can't see the forest for the trees

Every storage component in this codebase that uses CRC computes it **independently per record**. Each record carries its own checksum, computed solely from that record's payload. This means the CRC answers one question: *"Is this individual record's content intact?"* But it cannot answer: *"Is this record in the right place, in the right order, and are all the records that should be here actually here?"*

SQLite's WAL solves this with **chained frame checksums** — each frame's checksum is seeded with the previous frame's checksum, forming a hash chain. This turns a per-record integrity check into a *log-level* integrity check.

## How per-record CRCs work in this codebase

All three CRC-bearing components follow the same pattern. Take the standalone WAL (`write-ahead-log/wal.py:30-31`):

```python
crc_data = struct.pack("B", op_type_byte) + key + value
crc = zlib.crc32(crc_data) & 0xFFFFFFFF
```

On read (`wal.py:53-56`), the CRC is recomputed and compared:

```python
expected_crc = zlib.crc32(crc_data) & 0xFFFFFFFF
if crc != expected_crc:
    raise ValueError(f"CRC mismatch at seq {seq_num}")
```

The B-tree WAL (`btree.py:134-137`) does the same with page data:

```python
checksum = struct.pack('>I', self._checksum(page_data))
self._f.write(header + page_data + checksum)
```

Each record is a self-contained integrity unit. Pass or fail, the decision is made with zero awareness of neighboring records.

## The corruption classes that per-record CRCs miss

### 1. Record deletion (silent gap)

If a record is cleanly removed from the middle of a WAL file — the bytes before and after it are concatenated — every surviving record's CRC still passes. The log looks valid but is missing a mutation. With chained checksums, the record *after* the deleted one would fail verification because its checksum was seeded with the deleted record's checksum, which is now missing from the chain.

### 2. Record reordering

If two records swap positions on disk (due to a buggy rewrite, file system behavior, or an interrupted `truncate()`), both records individually validate. In this codebase, `truncate()` (`wal.py:104-110`) rewrites files in place — a crash during truncation could produce a valid-looking file with reordered records. Each record's CRC passes; the ordering corruption is invisible. A chained checksum would break at the first out-of-order frame.

### 3. Stale record splicing

Consider the B-tree WAL where each entry is a page image (`btree.py:120-137`). The CRC covers `page_data` but not `page_num`. During recovery (`btree.py:162-164`), a corrupted `page_num` writes a valid page to the *wrong location* — the checksum passes because the page data is intact. But even if `page_num` were included in the CRC, an attacker or bug could splice in a *complete valid record from an older WAL*. The record is internally consistent — correct CRC, correct headers — but it doesn't belong in this log at this position. A chained checksum would reject it because it breaks the chain.

### 4. Log truncation + extension

If a WAL file is truncated (losing its tail) and then new valid records are appended, per-record CRCs accept the entire file. The boundary between old and new data is invisible. In this codebase, `_read_record` (`wal.py:39-44`) treats a short read as benign EOF and `_recover_seq_num` (`wal.py:83-92`) simply finds the max sequence number — if the new records happen to have higher sequence numbers, the gap goes undetected. Chained checksums would fail at the splice point because the first new record was seeded with a different predecessor than what's actually on disk.

## How SQLite's chained checksums work

SQLite's WAL computes each frame's checksum by feeding the previous frame's checksum as the initial value:

```
frame[0].checksum = hash(frame[0].data, wal_header.salt)
frame[1].checksum = hash(frame[1].data, frame[0].checksum)
frame[2].checksum = hash(frame[2].data, frame[1].checksum)
...
```

This makes the entire log a single integrity unit. Verifying the last frame's checksum implicitly verifies every preceding frame. Any modification — deletion, insertion, reordering, splicing — anywhere in the chain invalidates all subsequent checksums.

## What this codebase would need to change

To add chaining to the standalone WAL, `_encode_record` would need to accept the previous record's checksum and fold it into the CRC computation:

```python
# Current (wal.py:30-31): independent per-record
crc_data = struct.pack("B", op_type_byte) + key + value
crc = zlib.crc32(crc_data) & 0xFFFFFFFF

# Chained: each record's CRC incorporates the previous
crc_data = struct.pack("B", op_type_byte) + key + value
crc = zlib.crc32(crc_data, prev_crc) & 0xFFFFFFFF  # zlib.crc32 accepts an initial value
```

The cost: you can no longer validate a single record in isolation. Recovery must start from the beginning of the chain. This is why SQLite's WAL always replays from the start — random access into the middle of the log requires scanning forward from the header.

## The practical gap in this codebase

The truncation rewrite in `wal.py:104-110` is the most dangerous interaction with the per-record CRC design. It rewrites files in place without atomic rename (documented in the entry as `wal-truncate-not-crash-safe`). A crash during truncation could produce a file where:

- The length-prefix framing is intact (layer 1 passes)
- Each surviving record's CRC is valid (layer 2 passes)
- But records are missing, reordered, or duplicated

None of these scenarios would be detected. Chained checksums would catch all of them.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_encode_record` — Modify this to accept a `prev_crc` parameter and observe how it changes the recovery and truncation code paths
- [function] `write-ahead-log/wal.py:truncate` — The most dangerous code in the WAL: rewrites files in place without atomicity, producing exactly the class of corruption that chained checksums detect
- [function] `b-tree-storage-engine/btree.py:WAL.recover` — A corrupted `page_num` passes the payload-only CRC and writes a valid page to the wrong disk location; chaining would not fix this specific gap (the `page_num` needs to be in the CRC input)
- [general] `sqlite-wal-checkpoint-semantics` — How SQLite's checkpoint mechanism interacts with the chained checksum to safely discard replayed frames
- [general] `rocksdb-block-framing-vs-chaining` — RocksDB uses fixed-size block framing with per-block CRCs rather than chaining; compare the tradeoffs (random access vs. whole-log integrity)

## Beliefs

- `per-record-crc-cannot-detect-reordering` — Independent per-record CRCs in `wal.py`, `btree.py`, and `bitcask.py` validate each record in isolation, so record deletion, reordering, or splicing produces a file where every CRC passes but the log is semantically corrupt
- `truncate-plus-per-record-crc-is-dangerous-combination` — The non-atomic `truncate()` in `wal.py:104-110` can produce reordered or incomplete files that pass per-record CRC validation; chained checksums would detect the corruption at the chain break point
- `zlib-crc32-supports-chaining-natively` — Python's `zlib.crc32(data, initial_value)` accepts an initial CRC value, meaning chained checksums could be implemented by passing the previous frame's CRC as the seed with no additional hashing infrastructure
- `chained-checksums-trade-random-access-for-log-integrity` — Chained frame checksums prevent mid-log splicing and reordering but require sequential validation from the chain start, eliminating the ability to validate or replay a single record in isolation

