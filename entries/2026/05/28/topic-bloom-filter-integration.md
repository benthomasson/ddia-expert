# Topic: Real LSM trees use bloom filters to skip SSTables that definitely don't contain a key; the `bloom-filter/` module in this repo is a natural extension

**Date:** 2026-05-28
**Time:** 18:27

# Bloom Filters: The Missing Optimization in LSM Tree Reads

## The Problem Bloom Filters Solve

When you look up a key in the LSM tree (`log-structured-merge-tree/lsm.py`), the engine must search each SSTable on disk until it finds the key or exhausts all tables. The `SSTable.get()` method (line 119) does a sparse index binary search followed by a sequential scan of the matching block. If the key isn't in that SSTable, you've done disk I/O for nothing.

This is the exact problem bloom filters address in production systems like LevelDB, RocksDB, and Cassandra: before touching the SSTable file at all, check a small in-memory bloom filter. If it says "definitely not here," skip the entire table.

## What the Bloom Filter Module Provides

The `bloom-filter/bloom_filter.py` implements three variants, each relevant to storage engines:

### Standard `BloomFilter` (lines 18–101)

The core probabilistic set. You configure it with `expected_items` and `false_positive_rate`, and it computes optimal bit array size (`_m`) and hash count (`_k`) using the standard formulas (lines 27–29):

```python
self._m = max(1, math.ceil(-expected_items * math.log(false_positive_rate) / (ln2 ** 2)))
self._k = max(1, round((self._m / expected_items) * ln2))
```

The `__contains__` method (lines 39–43) checks all `k` hash positions — if any bit is zero, the item is definitely absent. This is the "no false negatives" guarantee that makes bloom filters useful for SSTable skipping.

### How It Would Integrate

The integration point is `SSTable.get()` in both implementations. In `log-structured-merge-tree/lsm.py:119`, the method immediately jumps into sparse index lookup and file scanning. In a bloom-filter-enhanced version, you'd check the filter *before* any of that:

```python
def get(self, key):
    if self._bloom is not None and key not in self._bloom:
        return False, None  # definitely not here — skip disk I/O
    # ... existing sparse index + scan logic
```

The serialization support (`to_bytes`/`from_bytes`, lines 92–101) is specifically designed for this use case — you'd write the bloom filter bytes into the SSTable file (typically in the footer alongside the sparse index) and deserialize it when loading the table.

### Why the Repo Doesn't Integrate Them Yet

Grepping for "bloom" across both `log-structured-merge-tree/` and `sstable-and-compaction/` returns zero matches. The modules are taught as separate DDIA concepts. The bloom filter module is self-contained with its own test suite (`test_bloom_filter.py`, 207 lines covering false positive rates, serialization, unions, and edge cases), but it isn't wired into either SSTable implementation.

### The Other Variants

**`CountingBloomFilter`** (lines 104–140) replaces bits with counters, enabling `remove()`. This matters for compaction: when SSTables are merged and keys are dropped, the corresponding bloom filter entries need removal. The standard bloom filter can't do this — you'd have to rebuild it from scratch after compaction.

**`ScalableBloomFilter`** (lines 143–176) grows by adding filter slices with geometrically tightening FPR (`fp_ratio=0.5` per slice). This maps to the LSM memtable flush pattern: you don't know upfront how many keys an SSTable will hold, so a fixed-capacity filter risks either wasting memory or exceeding its target FPR.

## The Double Hashing Scheme

The `_hashes` function (lines 8–12) uses a single MD5 digest split into two 64-bit halves for double hashing: `(h1 + i * h2) % m`. This is the Kirschner-Mitzenmacher technique — it produces `k` independent-enough hash positions from one hash computation, which is critical for performance when you're checking filters on every read.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:SSTable.get` — Trace the full read path to see exactly where a bloom filter check would short-circuit disk I/O
- [function] `bloom-filter/bloom_filter.py:BloomFilter.to_bytes` — The serialization format that would embed into SSTable footers alongside the sparse index
- [function] `sstable-and-compaction/sstable.py:SSTableWriter.finish` — The footer-writing logic where bloom filter bytes would be appended during SSTable construction
- [general] `counting-bloom-compaction-interaction` — How CountingBloomFilter's `remove()` interacts with SSTable compaction (merge old tables → rebuild or update filter)
- [file] `sstable-and-compaction/sstable.py` — The more production-like SSTable with headers, timestamps, and tombstone markers — a better integration target than the simpler `lsm.py` version

## Beliefs

- `bloom-filter-not-integrated` — Neither LSM tree implementation (`lsm.py` or `sstable.py`) references or uses the bloom filter module; they exist as independent DDIA concept demonstrations
- `bloom-double-hashing` — `_hashes()` uses Kirschner-Mitzenmacher double hashing from a single MD5 digest, producing k positions as `(h1 + i*h2) % m` without k independent hash functions
- `bloom-serialization-footer-ready` — `BloomFilter.to_bytes` packs a 12-byte header (`m`, `k`, `count` as three little-endian uint32s) followed by the raw bit array, matching the pattern used by SSTable footers for sparse indices
- `counting-bloom-supports-deletion` — `CountingBloomFilter` uses saturating 4-bit counters (default) so entries can be removed, which standard `BloomFilter` cannot do — relevant for post-compaction filter updates
- `scalable-bloom-tightens-fpr` — Each new slice in `ScalableBloomFilter` uses FPR of `p * (ratio ^ slice_index)`, so the aggregate false positive rate stays bounded even as the filter grows unboundedly

