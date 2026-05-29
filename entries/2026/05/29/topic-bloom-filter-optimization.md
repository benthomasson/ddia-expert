# Topic: Production LSMs avoid touching SSTables entirely for missing keys using bloom filters; this complements the merge iterator by reducing k before the heap is even built

**Date:** 2026-05-29
**Time:** 07:33

# Bloom Filters and the Merge Iterator: Reducing k Before the Heap

## The Core Idea

When an LSM tree serves a point read, it must check multiple SSTables — the memtable first, then L0, L1, L2, etc. Without optimization, a **missing** key is the worst case: you search every SSTable before concluding the key doesn't exist. A bloom filter per SSTable lets you skip that table entirely with a single memory lookup, avoiding disk I/O for tables that definitely don't contain the key.

This directly reduces `k` in the merge iterator's k-way merge. If you have 10 SSTables but bloom filters eliminate 7 of them, you build a 3-element heap instead of a 10-element heap — O(log 3) per step instead of O(log 10), and more importantly, 7 fewer files opened.

## What This Codebase Shows

### The Bloom Filter (`bloom-filter/bloom_filter.py`)

The `BloomFilter` class at line 17 implements the classic structure: a bit array with `k` hash functions computed via double hashing (`_hashes`, line 8). The key operations are:

- **`add(item)`** (line 36): sets `k` bits in the array — called when an SSTable is built
- **`__contains__(item)`** (line 41): checks all `k` bits — if any bit is zero, the key is **definitely absent**. This is the fast path that avoids touching the SSTable at all.
- **`to_bytes()` / `from_bytes()`** (lines 91–103): serialization for embedding in SSTable file footers

The filter is parameterized by `expected_items` and `false_positive_rate` (line 19). The math at lines 26–28 computes optimal bit array size (`m`) and hash count (`k`) from these — the standard formulas from the Bloom (1970) paper.

### The SSTable Read Path (Without Bloom Filters)

In `log-structured-merge-tree/lsm.py`, `SSTable.get()` at line 117 does a sparse-index binary search then a linear scan within the block. For a missing key, this means:

1. Binary search the sparse index — O(log n) comparisons
2. Seek to the block offset — **disk I/O**
3. Scan entries until the key is passed — **more disk I/O**
4. Return `(False, None)`

The same pattern appears in `sstable-and-compaction/sstable.py`, where `SSTableReader.get()` (line 192) does a binary search over the sparse index then scans entries from the block.

### The Gap: No Integration

The observations reveal a critical architectural gap — **the bloom filter module is not integrated into either LSM implementation**:

- `lsm_bloom_usage`: 0 matches for `bloom|BloomFilter` in the LSM tree
- `sstable_bloom_usage`: 0 matches in the SSTable module
- `merge_iterator_pattern`: 0 matches for `merge|heap|iterator` in the bloom filter module

This means every point lookup in both SSTable implementations hits disk for the sparse index scan, even for keys that don't exist. In a production LSM (LevelDB, RocksDB, Cassandra), the read path would look like:

```
for sstable in sstables_newest_first:
    if key not in sstable.bloom_filter:  # ~10 bits of memory, no disk
        continue                          # skip entirely
    result = sstable.get(key)             # only now touch disk
    if result.found:
        return result
```

The bloom filter would be built during `SSTable.write()` (line 76 of `lsm.py`) by calling `bloom.add(key)` for each entry, then serialized into the file footer alongside the sparse index. On read, it would be loaded into memory (small — ~10 bits per key at 1% FPR) and checked before `get()` ever opens the data blocks.

### Why "Reducing k" Matters

The `heapq`-based merge pattern is visible in both files (`import heapq` at line 6 of `lsm.py` and `sstable.py`). During compaction or range scans, a k-way merge iterator pulls the smallest key from k SSTable iterators using a min-heap. But for **point lookups** the same principle applies at a higher level: you're effectively querying k sources. Each bloom filter that says "definitely not here" removes one source from consideration before any I/O happens. With a 1% FPR and 100 SSTables, you'd typically only touch 1–2 tables for a hit, and ~1 table (false positive) for a miss.

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:SSTable.write` — Where bloom filter construction should be wired in during SSTable flush
- [function] `bloom-filter/bloom_filter.py:BloomFilter.estimated_false_positive_rate` — The runtime FPR estimator that could drive compaction priority decisions
- [file] `sstable-and-compaction/sstable.py` — The compaction strategies (size-tiered, leveled) that determine how many SSTables exist and thus how much bloom filters save
- [general] `bloom-filter-integration` — Implementing the missing wiring: embed a bloom filter in each SSTable footer, load into memory on open, check before get()
- [function] `bloom-filter/bloom_filter.py:ScalableBloomFilter.__contains__` — The scalable variant checks all slices (line 172), showing how growth trades query speed for space flexibility

## Beliefs

- `bloom-filter-not-integrated` — The bloom filter module exists independently but is not used by either LSM/SSTable implementation; all point lookups always hit disk via the sparse index
- `bloom-filter-uses-double-hashing` — `_hashes()` derives k positions from a single MD5 digest via double hashing (h1 + i*h2) mod m, avoiding k independent hash calls
- `sstable-sparse-index-in-footer` — Both SSTable implementations store the sparse index at the end of the file with a footer pointer, meaning bloom filters could be co-located in the same footer region
- `bloom-serialization-exists` — `BloomFilter.to_bytes()`/`from_bytes()` provide a 12-byte header (m, k, count as little-endian u32) plus raw bit array, ready for embedding in SSTable files

