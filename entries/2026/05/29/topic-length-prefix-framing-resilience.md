# Topic: Whether any component can resync after a corrupted length field, or if all suffer cascading misframes

**Date:** 2026-05-29
**Time:** 12:05

I now have all the evidence needed. Here's the complete analysis:

---

# Resync After a Corrupted Length Field: None Can Recover

## The Short Answer

**No component in this codebase can resync after a corrupted length field.** Every binary record reader follows the same pattern: scan forward sequentially, stop the moment anything looks wrong, discard all subsequent records. There are no magic bytes, no block alignment, and no skip-ahead recovery. A single corrupted length field causes cascading misframes in every component that uses length-prefixed framing.

The one partial exception is the **event store**, which uses NDJSON (newline-delimited JSON) instead of binary framing — but even it stops on the first parse error rather than skipping the bad line.

## Component-by-Component Breakdown

### 1. WAL (`write-ahead-log/wal.py`) — Stop on CRC Error

The WAL has the **best corruption detection** in the codebase (CRC32), but still cannot resync.

Each record is framed as `[4B length][4B crc][8B seq_num][1B op_type][4B key_len][key][4B val_len][value]`. The reader at `wal.py:38-58` (`_read_record`) reads the 4-byte length prefix, then reads exactly that many bytes. If the length is corrupted:

- A **too-large** length causes a read that consumes valid subsequent records as "payload." The read returns fewer bytes than requested → `None` return → stop. All records after the corruption are lost.
- A **too-small** length causes the reader to land mid-record. The CRC check fails → `ValueError` → every caller breaks out of the loop (`wal.py:85-95`, `_recover_seq_num`).

The critical detail from `topic-record-boundary-recovery.md`: the CRC covers only `op_type + key + value` — it does **not** cover the length prefix itself. So the length field is the one unprotected link in the chain, and it's the one field that determines where every subsequent record boundary falls.

### 2. LSM WAL (`log-structured-merge-tree/lsm.py:14-53`) — Silent Cascading Corruption

This is the **worst case**. The LSM WAL uses bare length-prefixed fields (`struct.pack(">I", len(k))`) with **no CRC whatsoever**. The `replay()` method at lines 28-49 handles truncation (short reads return `None`) but has zero corruption detection.

A single bit flip in a length field causes the reader to jump to a garbage offset. From there, it interprets whatever bytes it lands on as the next key length, value length, key, and value — producing silently wrong data. There is no mechanism to detect this has happened, let alone recover from it. As noted in `topic-record-boundary-recovery.md`: "A single bit flip in length field → cascading silent data corruption during replay."

### 3. Bitcask with CRC (`log-structured-hash-table/bitcask.py`) — Stop on CRC Mismatch

The Bitcask `_scan_segment` function at lines 89-105 reads a fixed 12-byte header (`[4B crc][4B key_len][4B val_len]`), then reads key and value. If the CRC fails:

```python
if crc != expected_crc:
    break  # corrupted record at end, stop scanning
```

No attempt to skip forward. A corrupted `key_len` or `val_len` causes a misaligned read, which almost certainly produces a CRC mismatch — but the response is to abandon all remaining records in the segment. Segment rotation (`bitcask.py:118`) limits the blast radius to one file, but within that file, everything after the corruption is lost.

### 4. Bitcask without CRC (`hash-index-storage/bitcask.py`) — Blind Trust

The hash-index Bitcask uses a 16-byte header (`[8B timestamp][4B key_len][4B val_len]`) with **no CRC or checksum**. The `_scan_data_file` function at lines 119-136 reads the header, then reads key and value bytes based on the encoded lengths. The only guard is a short-read check on the header:

```python
if len(header_data) < HEADER_SIZE:
    break
```

A corrupted `key_len` or `val_len` that doesn't cause a short read produces **silent misframes** — the reader consumes wrong-sized chunks, misinterprets subsequent records, and loads garbage into the index. There is no detection, no recovery, and no error.

### 5. SSTable (`log-structured-merge-tree/lsm.py:72-196`) — Partial Isolation via Sparse Index

SSTables use per-entry framing (`[4B klen][key][4B vlen][value]`) with no CRC. A corrupted `klen` causes stream misalignment within a scan range. However, the **sparse index** provides a form of containment: `get()` only scans between two adjacent sparse index offsets (default 16 entries apart). A corrupted entry poisons that segment, but lookups for keys in other segments use clean offsets.

This is the closest thing to "partial resync" in the codebase — not because the reader can skip past corruption, but because the sparse index provides independent entry points into different regions of the file. The damage from `topic-sstable-footer-integrity.md` is bounded: "A corrupted data entry only affects lookups within one sparse index segment."

The **trailer** (last 12 bytes: `footer_start` + `count`) is an unprotected single point of failure. If the trailer is corrupted, the entire SSTable is unrecoverable — there's no redundant pointer or magic number to scan for.

### 6. Event Store (`event-sourcing-store/event_store.py`) — NDJSON Could Resync, But Doesn't

The event store uses **newline-delimited JSON**, which is the one format in this codebase where resync is architecturally possible. Each line is self-contained; a corrupted line doesn't affect the framing of subsequent lines because `\n` is the delimiter, not a length field.

However, `_load_from_file` does not exploit this. From the entry at `event-sourcing-store-event_store.md:108`: "If `_load_from_file` fails (disk full, corrupt JSON, permissions), the exception propagates directly to the caller." A `json.JSONDecodeError` on any line halts the entire load. The code could wrap each `json.loads` in a try/except and skip bad lines, but it doesn't.

### 7. B-tree (`b-tree-storage-engine/btree.py`) — Page-Level Isolation

The B-tree's `PageManager` uses **fixed-size pages** (default 4096 bytes), which provides natural resync boundaries — every page starts at `page_num * page_size`. This is conceptually similar to LevelDB's block alignment.

The WAL recovery at `btree.py`'s `WAL.recover` reads entries as `[4B seq][4B page_num][4B data_len][page_data][4B checksum]`. It uses CRC32 to gate replay: entries with mismatched checksums are silently skipped. But within the WAL itself, a corrupted `data_len` causes the same cascading misframe as everywhere else — the reader lands at a wrong offset and subsequent entries are lost (the bounds check at the header level catches this as a short read and breaks).

The key difference: the **data file** itself is page-aligned, so corruption of one page doesn't affect reads of other pages. The B-tree can read page 5 correctly even if page 3 is garbage. But this is a property of the page-based storage model, not of the WAL recovery mechanism.

## Why None of Them Can Resync

All seven components share a fundamental design choice: **records are packed contiguously with variable-length encoding, and the only way to find record N+1 is to correctly parse record N**. This creates a chain of trust anchored on the first byte of the file. Break any link — corrupt any length field — and every subsequent record boundary is unknown.

The contrast with LevelDB's design (described extensively in `topic-block-aligned-wal-records.md` and `topic-leveldb-log-format.md`) is instructive. LevelDB divides its log into fixed 32KB blocks. On corruption, the reader skips to `(position + 32768) & ~32767` — the next block boundary — and resumes scanning. You lose at most ~32KB of data instead of the entire tail. The `topic-record-boundary-recovery.md` entry summarizes the tradeoff: "block alignment wastes space at block boundaries...but [the WAL here] packs records tightly with zero waste."

For these reference implementations, stop-at-first-error is a reasonable choice: WAL files are short-lived and truncated after checkpoint, segment rotation bounds blast radius, and simplicity aids correctness in a teaching codebase. But none of them could survive mid-file corruption in a production setting without losing all subsequent records.

## Topics to Explore

- [general] `leveldb-block-alignment-tradeoffs` — Why LevelDB chose 32KB blocks and how the FIRST/MIDDLE/LAST fragment types enable skip-past-corruption recovery that this codebase lacks
- [function] `log-structured-merge-tree/lsm.py:WAL.replay` — The most dangerous reader: no CRC, no resync, silent corruption on any length-field bit flip — compare with the CRC-protected WAL in `write-ahead-log/wal.py:_read_record`
- [file] `event-sourcing-store/event_store.py` — The NDJSON format is the one place where newline-based resync is architecturally possible; explore what a try/except around `json.loads` in `_load_from_file` would buy
- [function] `b-tree-storage-engine/btree.py:PageManager.read_page` — Fixed-size pages provide natural alignment boundaries for the data file, unlike the streaming WAL formats; study how this isolation model compares to LevelDB's block-aligned logs
- [general] `crc-coverage-audit` — The WAL's CRC excludes the length prefix and seq_num; the Bitcask CRC excludes key_len/val_len; trace which fields are unprotected across all components and what corruption each misses

## Beliefs

- `no-component-resyncs-after-corruption` — Every binary record reader in this codebase stops at the first CRC failure, short read, or parse error; none attempt to scan forward for the next valid record boundary
- `lsm-wal-silent-cascading-misframes` — The LSM WAL (`lsm.py:14-53`) has no CRC and no resync; a single corrupted length field causes cascading silent data corruption through all subsequent records during replay
- `event-store-ndjson-could-resync-but-doesnt` — The event store's NDJSON format provides natural newline-based record boundaries that would allow skipping corrupt lines, but `_load_from_file` propagates `JSONDecodeError` instead of skipping
- `sstable-sparse-index-partial-isolation` — The SSTable's sparse index limits corruption blast radius to one segment (default 16 entries) by providing independent seek offsets, though this is containment, not resync
- `btree-page-alignment-provides-data-isolation` — The B-tree's fixed-size pages allow reading any page independently by arithmetic offset, isolating page-level corruption — but the WAL that protects the B-tree uses streaming records with no such alignment

