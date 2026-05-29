# Topic: Compare this record format with LevelDB's 32 KB block-aligned log format, which ensures torn writes always land at block boundaries and makes recovery O(block-size) instead of scanning from file start

**Date:** 2026-05-29
**Time:** 07:56

# WAL Record Format vs. LevelDB's Block-Aligned Log

## The Format in This Codebase: Length-Prefixed Streaming Records

Every WAL implementation here uses the same fundamental pattern: a **length-prefixed, variable-size record with no block alignment**.

The most complete example is `write-ahead-log/wal.py:29-34` (`_encode_record`):

```
┌──────────────┬──────────┬──────────┬─────────┬─────────┬─────┬───────────┬───────┐
│ record_length │   CRC32  │  seq_num │ op_type │ key_len │ key │ value_len │ value │
│   4 bytes     │  4 bytes │  8 bytes │ 1 byte  │ 4 bytes │ var │  4 bytes  │  var  │
└──────────────┴──────────┴──────────┴─────────┴─────────┴─────┴───────────┴───────┘
```

The reader (`wal.py:38-58`, `_read_record`) does: read 4-byte length, read that many bytes, unpack fields, verify CRC. If the length read is short or the CRC fails, the record is rejected.

The LSM tree's WAL (`log-structured-merge-tree/lsm.py:20-27`) is even more minimal — just `key_len | key | val_len | value` with no CRC at all. The Bitcask variants (`log-structured-hash-table/bitcask.py:10-12`) add CRC but keep the same streaming structure.

## What LevelDB Does Differently

LevelDB divides its log file into fixed **32 KB blocks** (32768 bytes). Every record lives within these boundaries:

```
Block (32 KB)
┌─────────────┬────────┬──────┬──────────┐
│  checksum   │ length │ type │   data   │
│   4 bytes   │ 2 bytes│1 byte│   var    │
├─────────────┼────────┼──────┼──────────┤
│  checksum   │ length │ type │   data   │
│   ...       │        │      │          │
├─────────────┴────────┴──────┴──────────┤
│         (padding to 32 KB)             │
└────────────────────────────────────────┘
```

The `type` field is one of `FULL`, `FIRST`, `MIDDLE`, `LAST`. A record that fits within the remaining space in a block gets type `FULL`. A record that would cross a block boundary is **split into fragments**: the first chunk gets `FIRST`, intermediate chunks get `MIDDLE`, and the final chunk gets `LAST`. Each fragment is independently checksummed.

## The Three Consequences

### 1. Recovery is O(block-size), not O(file-size)

Look at `wal.py:82-93` (`_recover_seq_num`):

```python
def _recover_seq_num(self) -> int:
    max_seq = 0
    for path in self._wal_files():
        with open(path, "rb") as f:
            while True:
                rec = _read_record(f)
                if rec is None:
                    break
                max_seq = max(max_seq, rec.seq_num)
    return max_seq
```

This scans **every record in every WAL file** from the start. A 10 MB WAL file (the default `max_file_size`) means reading 10 MB sequentially on startup. The LSM WAL replay (`lsm.py:28-49`) does the same — `f.read()` loads the entire file into memory and walks it byte-by-byte.

With LevelDB's block alignment, recovery can seek to the **last block boundary** (file size rounded down to the nearest 32 KB multiple) and scan just that one block. A torn write corrupts at most the last block, so you only need to validate ~32 KB regardless of file size. The rest of the file is either already checkpointed or can be verified block-by-block in parallel.

### 2. Torn writes are contained to block boundaries

In the streaming format here, a torn write (power loss mid-`write()`) produces a partial record anywhere in the file. The recovery code in `_read_record` (line 41-43) handles this by checking for short reads:

```python
record_data = f.read(record_length)
if len(record_data) < record_length:
    return None
```

This works, but it means the length prefix itself could be torn — you might read 2 of 4 length bytes and get a garbage length value, then try to read that many bytes, consuming valid records as "data." The code gets lucky here because a short read returns `None` and stops, but it can't distinguish between a clean EOF and a torn write followed by valid records.

LevelDB's block alignment guarantees that a torn write always falls within a single block. The block structure means you can always find the next valid block boundary by advancing to `(position + 32768) & ~32767`. Records in other blocks are unaffected.

### 3. Padding vs. packing tradeoff

The streaming format packs records tightly — no wasted space. LevelDB wastes up to 6 bytes of padding at the end of each block (if the remaining space is too small for even a header). For a workload of small records, this overhead is negligible. For large records that span many blocks, the per-fragment headers add ~7 bytes per 32 KB, also negligible. The real cost is complexity in the writer (splitting logic) and reader (reassembly logic).

## What This Means Practically

The streaming format in this codebase is simpler to implement and perfectly adequate for the file sizes involved (10 MB default in `wal.py:66`). The tradeoff becomes acute at scale: if you had a 1 GB WAL, recovery would scan 1 GB versus scanning 32 KB. The `checkpoint` method (`wal.py:163-170`) and `truncate` (`wal.py:172+`) mitigate this by keeping WAL files short, but they don't change the fundamental recovery complexity — it's still proportional to the size of the last un-truncated segment.

The Bitcask implementations face the same issue in `_scan_segment` (`log-structured-hash-table/bitcask.py:92-107`) and `_scan_data_file` (`hash-index-storage/bitcask.py:119-136`) — both scan from byte zero. Bitcask mitigates with hint files (a separate index), but that's solving a different problem (avoiding value reads during rebuild, not avoiding sequential scans).

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:append_batch` — Demonstrates the atomicity gap: the entire batch is written in one `write()` call with no block-boundary protection, so a torn write mid-batch silently loses the tail without corrupting earlier records
- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — Compare its CRC-based corruption detection with the WAL's short-read detection; this one stops scanning on CRC mismatch, silently dropping any valid records after the corrupt one
- [file] `b-tree-storage-engine/btree.py` — Contains a page-based WAL (line 120+) that uses fixed-size pages — a step toward block alignment, worth comparing with the streaming WAL
- [general] `leveldb-record-fragmentation` — Study how LevelDB's FIRST/MIDDLE/LAST fragment types handle records larger than 32 KB and what happens when a crash lands between fragments
- [function] `write-ahead-log/wal.py:truncate` — The mitigation strategy for O(n) recovery: keep WAL files short by discarding checkpointed records, trading write amplification for faster startup

## Beliefs

- `wal-recovery-scans-full-file` — `_recover_seq_num` in `write-ahead-log/wal.py` reads every record in every WAL file sequentially; there is no seek-to-end or block-skip optimization
- `lsm-wal-has-no-integrity-check` — The WAL in `log-structured-merge-tree/lsm.py` uses only length-prefix framing with no CRC or checksum; a single bit flip in the length field causes silent data corruption or misaligned reads
- `torn-length-prefix-causes-silent-skip` — If a torn write corrupts the 4-byte length prefix in `wal.py:_read_record`, the reader interprets garbage as `record_length`, reads that many bytes (consuming valid records), then returns `None` on short read — no error is raised
- `no-record-crosses-block-boundary` — None of the WAL implementations in this codebase use block-aligned or page-aligned record layouts; all records are packed contiguously with variable-length encoding

