# Topic: Standard (non-counting) bloom filters are the classic optimization for SSTable lookups in LSM trees (DDIA Chapter 3); explore why immutability makes the simpler filter sufficient there

**Date:** 2026-05-29
**Time:** 10:19

# Why Standard Bloom Filters Are Sufficient for SSTables

The core insight is a perfect alignment between a data structure's limitation and an architectural invariant: **standard bloom filters cannot handle deletions, and SSTables never delete individual entries**. This makes the simpler filter not just adequate but optimal.

## The Standard Filter's Limitation

Look at `BloomFilter` in `bloom-filter/bloom_filter.py:33-39`. The `add` method sets bits via bitwise OR:

```python
def add(self, item):
    for pos in _hashes(item, self._k, self._m):
        self._bits[pos // 8] |= 1 << (pos % 8)
    self._count += 1
```

Once a bit is set, there is no way to unset it. Multiple keys share hash positions (that's the source of false positives), so clearing a bit for one key could corrupt membership information for others. The `__contains__` check at line 41 simply tests whether all `k` bit positions are set — it has no concept of "this bit was set by which key."

## What Counting Filters Add (and What They Cost)

The `CountingBloomFilter` at `bloom-filter/bloom_filter.py:104-137` solves deletion by replacing each single bit with a multi-bit counter:

```python
self._counters = bytearray(self._m)  # one byte per counter
```

Where the standard filter uses one *bit* per position, the counting variant uses one *byte* (or more generally, `counter_bits` bits). That's an **8x space increase** at minimum. The `remove` method at line 121 can safely decrement counters because each addition increments them — but it must also handle saturation (line 127: saturated counters stay fixed to avoid underflow corruption).

This space penalty matters enormously for bloom filters attached to SSTables. A production LSM tree might have hundreds of SSTables on disk, each with its own filter. An 8x memory overhead *per filter* across all levels of the LSM tree is a serious cost.

## Why SSTables Never Need Deletion

The key architectural fact is visible in `log-structured-merge-tree/lsm.py:73-99` and `sstable-and-compaction/sstable.py:50-100`. SSTables are **write-once, read-many**:

1. **Creation**: `SSTable.write()` (lsm.py:78) takes a sorted list of entries and writes them sequentially to a file. After `write` returns, the file is sealed.
2. **No mutation API**: Neither `SSTable` nor `SSTableReader` exposes any method to modify entries in place. There is no `delete`, `update`, or `remove` method on the SSTable class.
3. **Deletes are tombstones**: When a key is deleted in the LSM tree, it writes a `TOMBSTONE` marker (`lsm.py:10`: `TOMBSTONE = b""`) as a *new entry* — it doesn't go back and remove the key from existing SSTables.
4. **Compaction replaces whole files**: Old SSTables are eliminated by compaction, which reads entries from multiple SSTables and writes entirely new ones. The old files are then deleted wholesale.

So the lifecycle of an SSTable's bloom filter is:

1. **Build** — during SSTable creation, add every key
2. **Query** — during reads, check membership to skip disk I/O
3. **Discard** — when the SSTable is compacted away, discard the filter entirely

There is never a step where a single key needs to be removed from the filter while the rest remain. The filter lives and dies with its SSTable.

## The Missing Integration

Notably, this codebase does not yet wire bloom filters into the SSTable or LSM tree implementations — grepping for `bloom` or `BloomFilter` in both `log-structured-merge-tree/` and `sstable-and-compaction/` returns zero matches. The bloom filter exists as a standalone module. In a production integration, each `SSTable` would hold a `BloomFilter` built during `SSTable.write()`, and `SSTable.get()` (lsm.py:116) would check the filter *before* performing the sparse index binary search and disk scan — avoiding disk I/O entirely for keys that aren't present.

The serialization support at `bloom-filter/bloom_filter.py:92-103` (`to_bytes`/`from_bytes`) is exactly what this integration would need: the filter's bit array would be stored in the SSTable file (likely in the footer alongside the sparse index) and loaded when the SSTable is opened.

## Topics to Explore

- [function] `bloom-filter/bloom_filter.py:_hashes` — Double-hashing scheme that generates k positions from a single MD5 digest; worth understanding how this avoids k independent hash functions
- [function] `log-structured-merge-tree/lsm.py:SSTable.get` — The sparse-index lookup that a bloom filter would short-circuit; trace the binary search + scan path to see what I/O is saved
- [general] `bloom-filter-sstable-integration` — Design how BloomFilter would be embedded in the SSTable footer and checked during reads — the natural next implementation step
- [function] `bloom-filter/bloom_filter.py:CountingBloomFilter.remove` — The saturation guard on line 127 is a subtle correctness issue; explore what happens when counters overflow and why it makes false negatives possible
- [general] `compaction-filter-rebuild` — During size-tiered or leveled compaction, new SSTables get new bloom filters; explore how filter parameters (m, k) should scale with SSTable size across levels

## Beliefs

- `standard-bloom-suffices-for-immutable-sstables` — Standard bloom filters lack deletion support, but SSTables are write-once-then-discarded, so deletion is never needed and the 8x space savings over counting filters is free
- `counting-bloom-8x-overhead` — CountingBloomFilter uses one byte per counter position vs one bit in BloomFilter, an 8x minimum space cost to support removal
- `bloom-filter-not-integrated-in-lsm` — Neither the LSM tree nor SSTable implementations import or use BloomFilter; the filter exists as a standalone module without SSTable integration
- `bloom-serialization-enables-sstable-embedding` — BloomFilter.to_bytes/from_bytes provide the serialization needed to embed filters in SSTable files, storing m, k, count, and the bit array in a 12-byte header plus the bit array

