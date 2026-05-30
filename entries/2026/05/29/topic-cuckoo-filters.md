# Topic: The modern alternative to counting Bloom filters that supports deletion with better space efficiency; worth comparing trade-offs

**Date:** 2026-05-29
**Time:** 10:17

# Cuckoo Filters: The Modern Alternative to Counting Bloom Filters

## What the Codebase Has Today

The repository implements a `CountingBloomFilter` at `bloom-filter/bloom_filter.py:104-138`. It supports deletion by replacing each single bit with a multi-bit counter — when you add an item, counters increment; when you remove, they decrement. The critical trade-off is visible on line 112:

```python
self._counters = bytearray(self._m)  # one byte per counter for simplicity
```

Even though `counter_bits=4` is the default (line 106), each counter occupies a full byte. A standard Bloom filter uses 1 bit per position; a counting Bloom filter with 4-bit counters uses **4x the space**, and this implementation actually uses **8x** for simplicity. That's the cost of supporting deletion.

The saturation behavior on lines 117-119 and 126-127 reveals another weakness: once a counter hits `_max_val` (15 for 4-bit counters), it stays pinned. Saturated counters are never decremented during removal, which means heavy-traffic positions silently lose the ability to delete. The test at `bloom-filter/test_bloom_filter.py:86-93` (`test_counting_double_add_remove`) validates the happy path but doesn't exercise saturation.

## The Cuckoo Filter Alternative

A **cuckoo filter** solves the same problem — probabilistic membership with deletion support — using a fundamentally different mechanism. Instead of hashing to bit/counter positions, it:

1. **Stores fingerprints** (small hashes, typically 8-12 bits) in a hash table with cuckoo hashing
2. **Uses two candidate buckets** per item (derived from the item's hash and its fingerprint via partial-key cuckoo hashing)
3. **Relocates existing entries** when both buckets are full, evicting an occupant to its alternate bucket — the "cuckoo" operation

Deletion is straightforward: find the matching fingerprint in one of the two candidate buckets and remove it. No counters, no saturation problem.

## Trade-off Comparison

| Dimension | CountingBloomFilter (this repo) | Cuckoo Filter |
|---|---|---|
| **Space per element** | ~4-8x a standard Bloom filter | ~1.05-1.2x a standard Bloom filter at similar FPR |
| **Deletion** | Yes, but saturated counters break it (line 126) | Yes, clean — remove fingerprint from bucket |
| **Lookup** | k hash computations, k array reads | 2 bucket lookups (constant, cache-friendly) |
| **Insertion worst case** | O(k), always succeeds | May require chain of relocations; can fail if table is too full (>95% load) |
| **False positive source** | Hash collisions across k functions | Fingerprint collisions (shorter fingerprint = higher FPR) |
| **Union/merge** | Not supported (counters don't OR) | Not naturally supported either |
| **Duplicate inserts** | Handled correctly — counters increment (tested at line 86) | Limited — most implementations cap duplicates at 2kb (bucket size × 2) |

## Why It Matters for DDIA

The counting Bloom filter in this repo uses the same `_hashes` function (`bloom_filter.py:9-14`) as the standard filter — double hashing with MD5. A cuckoo filter would replace this entirely with a fingerprint-and-bucket scheme, which is architecturally different enough that it wouldn't share the `_hashes` helper.

The space advantage is significant in storage engines. When an LSM-tree attaches a Bloom filter to each SSTable (as this repository's `sstable-and-compaction/` module does), supporting deletion without the 4-8x space overhead means you can afford filters on more levels or with lower FPR targets.

## What's Missing from the Observations

The codebase has **no cuckoo filter implementation**. The searches for `cuckoo`, `fingerprint`, `relocat`, and `evict` found nothing relevant (only SSTable compaction buckets, which are unrelated). This is a natural extension point — implementing a `CuckooFilter` class with the same `add`/`remove`/`__contains__` interface as `CountingBloomFilter` would allow direct benchmarking of the space/performance trade-offs described above.

---

## Topics to Explore

- [general] `cuckoo-filter-implementation` — Implement a CuckooFilter class with the same interface as CountingBloomFilter to benchmark space efficiency and FPR empirically
- [function] `bloom-filter/bloom_filter.py:_hashes` — The double-hashing scheme; compare with partial-key cuckoo hashing where the alternate bucket is derived from `bucket XOR hash(fingerprint)`
- [function] `bloom-filter/bloom_filter.py:CountingBloomFilter.remove` — Study the saturation edge case on line 126 where counters pinned at max silently skip decrement, creating a leak
- [file] `sstable-and-compaction/sstable.py` — Where Bloom filters integrate with the storage engine; evaluate whether cuckoo filters' deletion support enables filter updates during compaction without rebuild
- [general] `xor-filters` — Even newer than cuckoo filters; static (no insertion after build) but achieve near-theoretical-minimum space — relevant for immutable SSTables where post-build deletion isn't needed

## Beliefs

- `counting-bloom-counter-saturation-breaks-delete` — When a CountingBloomFilter counter reaches `_max_val` (default 15), it pins and is never decremented on remove, silently breaking deletion correctness for that position
- `counting-bloom-8x-space-overhead` — The CountingBloomFilter implementation uses one byte per counter position (`bytearray(self._m)` at line 112), 8x the space of a standard BloomFilter's bit array, despite the `counter_bits` parameter suggesting 4-bit counters
- `no-cuckoo-filter-in-repo` — The repository has no cuckoo filter, quotient filter, or xor filter implementation; CountingBloomFilter is the only deletion-capable probabilistic membership structure
- `bloom-and-counting-share-hash-function` — Both BloomFilter and CountingBloomFilter use the same `_hashes()` double-hashing scheme (MD5-based, lines 9-14), meaning their bit/counter positions are identically distributed for the same parameters

