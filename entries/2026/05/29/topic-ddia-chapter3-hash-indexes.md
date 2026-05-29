# Topic: DDIA Chapter 3's hash index section motivates this entire design and discusses the tradeoffs vs. SSTables/LSM trees

**Date:** 2026-05-29
**Time:** 08:43

# Hash Indexes vs. SSTables/LSM Trees in This Codebase

## The Core Idea from DDIA Chapter 3

Kleppmann opens Chapter 3 with the simplest possible database — an append-only log — and asks: how do you find things in it without scanning the whole file? The answer is a **hash index**: keep an in-memory dictionary mapping every key to its byte offset on disk. This is exactly what Bitcask (Riak's storage engine) does, and it's exactly what two implementations in this repo demonstrate.

The chapter then asks: what are the *limitations* of this approach, and what do you gain by switching to **sorted structures** (SSTables, LSM trees)? The remaining two implementations answer that question.

## The Hash Index Implementations

Both `hash-index-storage/bitcask.py` and `log-structured-hash-table/bitcask.py` implement the same fundamental contract from DDIA:

**Writes are always sequential appends.** In `hash-index-storage/bitcask.py:88-96`, `_write_record` appends a header + key + value to the active file and calls `fsync`. In `log-structured-hash-table/bitcask.py:137-145`, `_write_record` does the same but adds a CRC32 checksum for integrity. Neither implementation ever overwrites data in place.

**Reads are one index lookup + one disk seek.** The `keydir` dictionary (`hash-index-storage/bitcask.py:36`) maps every key to a `KeyEntry(file_id, offset, size, timestamp)`. A `get()` call (line 165) does `self.keydir.get(key)` then `self._read_record(entry.file_id, entry.offset, entry.size)`. This is the O(1) lookup DDIA describes — exactly one hash table lookup, exactly one disk read. The second implementation (`log-structured-hash-table/bitcask.py:44`) stores `(filepath, offset)` tuples instead, but the principle is identical.

**Deletes use tombstones.** Both implementations handle deletion by appending a special record — an empty-string value (`hash-index-storage/bitcask.py:176`) or a `TOMBSTONE` sentinel (`log-structured-hash-table/bitcask.py:14`). The key is removed from the in-memory index immediately, but the old data remains on disk until compaction.

**Segment rotation prevents unbounded file growth.** When the active file exceeds `max_file_size`, both rotate to a new segment (`hash-index-storage/bitcask.py:107-110`, `log-structured-hash-table/bitcask.py:127-130`). Old segments become immutable and eligible for compaction — this is the "break the log into segments" strategy DDIA describes.

**Hint files accelerate startup.** On crash recovery, the naive approach is scanning every data file to rebuild the index (`hash-index-storage/bitcask.py:120-137`). Hint files (`_write_hint_file` at line 152, `_load_hint_file` at line 139) store just the key-to-offset mappings without the values, so recovery reads far less data. This is a direct implementation of the Bitcask hint file optimization Kleppmann mentions.

## The Tradeoff: Why SSTables and LSM Trees?

DDIA identifies three fundamental limitations of hash indexes that motivate the move to sorted structures:

### 1. The index must fit in memory

The hash index stores **every key** in RAM. The `keydir` dict in `hash-index-storage/bitcask.py:36` grows linearly with the number of distinct keys. If you have a billion keys, you need a billion hash entries in memory. There is no way to spill part of the index to disk efficiently — a hash table on disk requires random I/O for every lookup.

The LSM tree (`log-structured-merge-tree/lsm.py`) solves this with a **sparse index**. The `SSTable` class (line 72) stores entries sorted by key on disk and keeps only every 16th key in memory (`sparse_index_interval=16`, line 74). A lookup does a binary search in the sparse index to find the right block, then scans a small range. The SSTable in `sstable-and-compaction/sstable.py:52` does the same with configurable `block_size=64`. This means an SSTable with a million entries only needs ~62,500 index entries in memory (or ~15,625 with block size 64).

### 2. No efficient range queries

Look at the grep results: searching for `range|scan|sorted|order` across the hash index implementations returns **zero matches**. This is the design limitation DDIA emphasizes — hash indexes can only answer "what is the value for key X?" They cannot efficiently answer "give me all keys between `sensor:temp:100` and `sensor:temp:200`."

The SSTable implementations provide exactly this. `log-structured-merge-tree/lsm.py:146-163` implements `scan(start_key, end_key)` which seeks to the right position using the sparse index and then reads sequentially. Because entries are sorted on disk, a range scan is a sequential read — the best possible I/O pattern. The `sstable-and-compaction/sstable.py` reader provides the same capability.

### 3. Compaction is simpler with sorted data

Both hash-index implementations must do compaction (merge old segments, discard superseded values). But the LSM tree gets a structural advantage: because SSTables are sorted, merging is a **merge-sort** operation — you walk multiple sorted files in parallel and write one sorted output. This is the `heapq` import in `log-structured-merge-tree/lsm.py:6` and `sstable-and-compaction/sstable.py:2`. The hash index compaction, by contrast, must check every record against the current keydir to decide if it's still live.

## What the Hash Index Wins

DDIA is clear that hash indexes aren't simply worse — they have genuine advantages for the right workload:

- **Simpler implementation.** Compare the two Bitcask files (~320-390 lines each) with the LSM tree (~360 lines for `lsm.py` alone, plus ~440 lines for the SSTable layer). The hash index has fewer moving parts — no memtable, no WAL, no merge-sort compaction, no bloom filters.
- **Predictable read latency.** A hash index read is always one hash lookup + one disk seek. An LSM tree read might check the memtable, then multiple SSTable levels, each requiring a sparse-index search + scan.
- **Fast writes.** Both are append-only, but the hash index doesn't need to maintain sort order in memory (no `SortedDict` like `lsm.py:8`). Writes are pure appends with an index update.
- **CRC integrity checking.** The `log-structured-hash-table/bitcask.py:19-20` variant adds per-record CRC32 validation, which catches corruption at read time (line 175-178). The simpler hash index in `hash-index-storage/bitcask.py` omits this — a design point Kleppmann notes as important for real deployments.

## The LSM Tree's Additional Machinery

The LSM tree requires infrastructure the hash index doesn't:

- **Write-ahead log** (`log-structured-merge-tree/lsm.py:14-64`): Because the memtable is in-memory and sorted, you need a WAL to survive crashes. The hash index writes directly to the data file, which *is* the log.
- **Memtable** (a `SortedDict`): Buffers writes in sorted order before flushing to an SSTable. This is the component that enables sorted output.
- **Multi-level compaction**: The `sstable-and-compaction/sstable.py` file implements both size-tiered and leveled compaction — two strategies DDIA discusses in detail for managing read amplification vs. write amplification.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — The compaction logic (lines 183+) is cut off in the observation; understanding how it differs from SSTable merge-sort compaction is the crux of the tradeoff
- [file] `log-structured-merge-tree/lsm.py` — Lines 200+ contain the LSMTree class itself, including memtable flush, multi-level lookup, and compaction orchestration
- [file] `sstable-and-compaction/sstable.py` — Lines 200+ likely contain the compaction strategy implementations (size-tiered vs. leveled) that DDIA Chapter 3 contrasts
- [general] `bloom-filter-optimization` — DDIA discusses bloom filters as essential for LSM read performance (avoiding unnecessary SSTable scans); check whether this repo implements one
- [function] `log-structured-hash-table/bitcask.py:compact` — Compare the CRC-validated variant's compaction with the simpler hash-index compaction to see how integrity checking composes with segment merging

## Beliefs

- `hash-index-all-keys-in-memory` — Both Bitcask implementations keep every live key in an in-memory dict (`keydir` / `_index`), meaning the key set must fit in RAM; there is no disk-based fallback
- `sstable-sparse-index-reduces-memory` — The SSTable sparse index stores one entry per N records (default 16 in `lsm.py`, 64 in `sstable.py`), reducing memory usage by that factor compared to a hash index
- `hash-index-no-range-queries` — Neither hash index implementation provides a `scan`, `range`, or ordered-iteration method; only point lookups by exact key are supported
- `lsm-requires-wal-hash-does-not` — The LSM tree uses a separate WAL (`lsm.py:14-64`) for crash recovery because the memtable is volatile; the hash index writes directly to the append-only data file, which serves as its own log
- `crc-integrity-only-in-log-structured-variant` — Only `log-structured-hash-table/bitcask.py` validates records with CRC32 checksums; `hash-index-storage/bitcask.py` trusts disk reads without integrity verification

