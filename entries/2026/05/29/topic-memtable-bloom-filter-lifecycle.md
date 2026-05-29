# Topic: The one scenario where `CountingBloomFilter.remove()` is genuinely useful: maintaining a filter over a mutable in-memory table where keys can be overwritten or deleted without a full rescan

**Date:** 2026-05-29
**Time:** 07:35

## When `CountingBloomFilter.remove()` Actually Earns Its Keep

A standard `BloomFilter` (`bloom_filter.py:17`) is a one-way door: bits flip to 1 and never go back. That's fine for immutable data — once an SSTable is written to disk, its contents are frozen. But what about a data structure whose contents change while the filter is live?

### The Problem: Mutable Tables Invalidate Standard Bloom Filters

Look at the LSM tree's memtable in `lsm.py:209`:

```python
self._memtable: SortedDict = SortedDict()
```

This is a mutable in-memory sorted dictionary. Keys can be **overwritten** (`lsm.py:232`, `put` writes the same key again) and **deleted** (`lsm.py:268`, delete writes a `TOMBSTONE`). If you placed a standard `BloomFilter` in front of this memtable to skip unnecessary lookups, it would go stale the moment you delete a key — the bit positions for that key stay set, but the key is no longer logically present. Over time, the filter's false positive rate climbs toward uselessness. Your only recovery option is to rebuild the entire filter by rescanning every key in the memtable.

### The Solution: Counters Instead of Bits

`CountingBloomFilter` (`bloom_filter.py:98`) replaces single bits with integer counters. When `add()` is called (`bloom_filter.py:108`), counters increment. When `remove()` is called (`bloom_filter.py:113`), they decrement. Membership is checked by testing whether all counters are positive (`bloom_filter.py:123`):

```python
def __contains__(self, item):
    return all(self._counters[pos] > 0 for pos in _hashes(item, self._k, self._m))
```

This means the filter can track a mutable set of keys. When the memtable deletes key `"user:42"`, you call `cbf.remove("user:42")`, and the filter accurately reflects that the key is gone — no full rescan needed.

### Why This Is the *One* Good Scenario

The reason this is narrowly useful comes down to what the counters cost and what they buy:

**Cost:** Each position uses a full byte (`bloom_filter.py:106`, `bytearray(self._m)`) instead of a single bit. That's 8x the memory of a standard bloom filter. With 4-bit counters (the default `counter_bits=4`), the max counter value is 15 (`bloom_filter.py:105`, `(1 << counter_bits) - 1`). Beyond that, counters saturate and **cannot be decremented** (`bloom_filter.py:119`):

```python
if self._counters[pos] < self._max_val:  # saturated counters stay
    self._counters[pos] -= 1
```

This saturation guard is critical — decrementing a saturated counter could create false negatives, which bloom filters must never produce.

**What they buy:** Incremental updates to the filter as the underlying data mutates. This only matters when:
1. The data is **mutable** (keys come and go) — immutable SSTables don't need this.
2. The data is **in-memory** — rebuilding a filter from an on-disk table is I/O-expensive, but rescanning an in-memory `SortedDict` is cheap enough that a counting filter is often not worth the 8x memory overhead.
3. Mutations are **frequent** relative to the table size — if deletes are rare, an occasional full rebuild is simpler.

The memtable sits right at this intersection: it's mutable, it's in memory, and in write-heavy workloads it sees constant churn. Notice that the LSM tree currently does **not** attach any bloom filter to its memtable (grepping for `bloom` or `BloomFilter` in `lsm.py` returns zero hits). The standard bloom filter is the right choice for SSTables (immutable on disk), while a counting filter would be the right choice for the memtable — if the lookup savings justify the memory cost.

### The Test Confirms the Contract

The test at `test_bloom_filter.py:80` demonstrates the key invariant — double-add followed by single-remove leaves the item present:

```python
cbf.add("x")
cbf.add("x")
cbf.remove("x")
assert "x" in cbf  # still present after one removal
cbf.remove("x")
assert "x" not in cbf
```

This mirrors exactly what happens when a memtable key is overwritten then deleted: each operation must be individually tracked.

---

## Topics to Explore

- [file] `log-structured-merge-tree/lsm.py` — The LSM memtable is the natural consumer for a counting bloom filter; trace how `put`, `get`, and `delete` mutate the memtable to see where filter checks would be inserted
- [function] `bloom-filter/bloom_filter.py:remove` — Study the saturation guard at line 119; try to construct a scenario where removing a saturated counter would introduce a false negative
- [general] `bloom-filter-on-sstables` — Standard (non-counting) bloom filters are the classic optimization for SSTable lookups in LSM trees (DDIA Chapter 3); explore why immutability makes the simpler filter sufficient there
- [function] `log-structured-merge-tree/lsm.py:_flush` — When the memtable flushes to an SSTable, any attached counting bloom filter would be discarded; understand this lifecycle boundary
- [general] `counter-overflow-and-cuckoo-filters` — The 4-bit saturation limit is a known weakness of counting bloom filters; cuckoo filters solve the same problem with better space efficiency and perfect deletion support

## Beliefs

- `cbf-saturated-counters-are-permanent` — When a `CountingBloomFilter` counter reaches `_max_val` (default 15), it is never decremented, preventing false negatives at the cost of a slight false positive rate increase
- `cbf-remove-raises-on-absent-item` — `CountingBloomFilter.remove()` raises `ValueError` if any hash position has a zero counter, providing a safety check against removing items that were never added
- `lsm-memtable-has-no-bloom-filter` — The LSM tree implementation does not attach any bloom filter (standard or counting) to its memtable, meaning every `get()` performs a direct dictionary lookup
- `cbf-8x-memory-vs-standard` — `CountingBloomFilter` uses one byte per counter position (`bytearray(self._m)`) versus one bit per position in `BloomFilter`, an 8x memory overhead for deletion support

