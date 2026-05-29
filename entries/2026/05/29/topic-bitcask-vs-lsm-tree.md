# Topic: How this hash-index approach (O(1) reads, all keys in memory) compares to the LSM-tree in `log-structured-merge-tree/`, which trades write amplification for range query support

**Date:** 2026-05-29
**Time:** 08:38

# Hash Index vs. LSM-Tree: A Side-by-Side Comparison

## The Core Trade-off

Both engines are **append-only log-structured stores**, but they make fundamentally different choices about what to keep in memory and how data is organized on disk.

### Hash Index (Bitcask): O(1) Point Lookups, No Range Queries

The hash index in `hash-index-storage/bitcask.py` keeps a complete **in-memory dictionary** mapping every key to its disk location:

```python
# bitcask.py:37 — the entire keydir lives in RAM
self.keydir: dict[str, KeyEntry] = {}
```

A `get()` (line 155) is a dictionary lookup followed by a single disk seek — constant time regardless of dataset size. A `put()` (line 150) appends to the active file and updates the hash map. There's no ordering of keys anywhere — the `keys()` method (line 167) just dumps the dictionary.

The second Bitcask variant in `log-structured-hash-table/bitcask.py` is structurally identical but adds CRC32 integrity checks (`HEADER_FMT = "!III"`, line 13) and a `CorruptionError` (line 19). Both variants share the same fundamental limitation: **every key must fit in memory**, and there is no way to answer "give me all keys between A and D" without scanning the entire hash map.

### LSM-Tree: Sorted Structure Enables Range Queries

The LSM-tree in `log-structured-merge-tree/lsm.py` uses a `SortedDict` as its memtable (line 8: `from sortedcontainers import SortedDict`). When the memtable fills up, it's flushed as a **sorted SSTable** — an on-disk file where keys are in lexicographic order with a sparse index for binary search.

This sorted structure enables `range_scan()`, which the test at `test_lsm.py:150-153` exercises:

```python
results = db.range_scan("a", "e")
assert results == [("a", "1"), ("b", "updated"), ("c", "3"), ("d", "4")]
```

Range scans merge results from the memtable and all SSTables using a heap-based merge (the `heapq` import at `lsm.py:5`). The hash index has **no equivalent operation** — grep for `range|scan|iterator` across `hash-index-storage/` returned zero matches.

## Detailed Comparison

| Dimension | Hash Index (Bitcask) | LSM-Tree |
|-----------|---------------------|----------|
| **Read path** | Hash lookup → 1 disk seek (O(1)) | Memtable → SSTables newest-first (O(log N) per level) |
| **Write path** | Append + update hash map | Append to WAL + insert into sorted memtable |
| **Range queries** | Not supported | Native via sorted merge across memtable + SSTables |
| **Memory requirement** | All keys must fit in RAM | Only memtable + sparse indexes in RAM |
| **Crash recovery** | Scan all data files or load hint files (`_rebuild_index`, bitcask.py:107) | Replay WAL (`WAL.replay()`, lsm.py:31) — only unflushed writes |
| **Write amplification** | Low — compaction rewrites each live value once | Higher — values may be rewritten across multiple compaction levels |

## Recovery Cost Differences

The Bitcask recovery at `bitcask.py:107-114` must scan every data file on startup unless hint files exist. Hint files (`_write_hint_file`, line 141) are an optimization written during compaction that store just the key-to-offset mappings, avoiding a full data scan.

The LSM-tree uses a WAL (`lsm.py:11-60`) that's replayed on startup. Only unflushed memtable entries need recovery — everything already flushed to SSTables is durable by construction. The test at `test_lsm.py:96-103` verifies this: it opens a new `LSMTree` without closing the first, and the WAL replay restores `x` and `y`.

## The SSTable Building Block

The `sstable-and-compaction/sstable.py` module provides a more production-grade SSTable implementation with magic bytes (`MAGIC = b"SSTB"`, line 10), explicit tombstone markers (line 12), and both writer/reader classes. This is the same sorted-file concept the LSM-tree depends on, but extracted as a reusable component with block-level sparse indexes (`block_size=64`, line 48) for efficient binary search within files.

## When to Choose Which

The hash index wins when your workload is **all point lookups and updates** — think session stores, caches, or sensor readings keyed by device ID. The LSM-tree wins when you need **range queries or can't guarantee all keys fit in RAM** — time-series data, ordered event logs, or anything where you query by key prefix or range.

---

## Topics to Explore

- [file] `log-structured-merge-tree/lsm.py` (lines 200-359) — The `LSMTree` class itself: memtable flush logic, multi-SSTable reads, compaction strategy, and the `range_scan` merge algorithm are all in the unshown second half
- [function] `sstable-and-compaction/sstable.py:SSTableReader.get` — Binary search over sparse index followed by linear scan within a block; compare this O(log N + block) lookup cost against Bitcask's O(1) hash lookup
- [function] `hash-index-storage/bitcask.py:compact` (line 170+) — The compaction strategy for Bitcask: how it merges segments while keeping only the latest value per key, and how it writes hint files for fast recovery
- [general] `write-amplification-measurement` — Instrument both engines to count total bytes written per logical put, especially under update-heavy workloads with compaction — this quantifies the LSM-tree's main cost
- [diff] `log-structured-hash-table/bitcask.py` vs `hash-index-storage/bitcask.py` — Two Bitcask implementations with different integrity guarantees (CRC32 vs. none) and API surfaces (bytes vs. strings); compare their design choices

## Beliefs

- `hash-index-keys-must-fit-in-ram` — Both Bitcask implementations require every live key to be held in a Python dict in memory; the dataset's key space is bounded by available RAM
- `lsm-tree-supports-range-scans-bitcask-does-not` — The LSM-tree provides a `range_scan` method that merges sorted results from memtable and SSTables; neither Bitcask implementation has any range or ordered-iteration capability
- `bitcask-recovery-requires-full-scan-without-hints` — On startup without hint files, Bitcask must sequentially read every record in every data file to rebuild the in-memory index (`_rebuild_index` → `_scan_data_file`)
- `lsm-wal-limits-recovery-to-unflushed-writes` — The LSM-tree's WAL replay only needs to recover entries written since the last memtable flush; all flushed data is already durable in SSTables
- `hash-index-read-is-single-seek` — A Bitcask `get()` does one dict lookup plus one positioned disk read, while an LSM-tree `get()` may search the memtable and then multiple SSTables from newest to oldest

