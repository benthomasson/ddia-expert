# Topic: The 4-bit saturation limit is a known weakness of counting bloom filters; cuckoo filters solve the same problem with better space efficiency and perfect deletion support

**Date:** 2026-05-29
**Time:** 10:19

# The 4-Bit Saturation Problem in Counting Bloom Filters

## The Problem, Visible in Code

The `CountingBloomFilter` in `bloom-filter/bloom_filter.py` replaces single bits with counters so that items can be deleted — but this introduces a subtle correctness hazard at the counter ceiling.

At line 107, the counter width defaults to 4 bits:

```python
def __init__(self, expected_items=1000, false_positive_rate=0.01,
             counter_bits=4):
```

This means `_max_val` is `(1 << 4) - 1 = 15` (line 110). A counter can only count to 15 before it **saturates** — it stops incrementing and becomes permanently stuck.

The `add` method (line 111–114) enforces this ceiling:

```python
def add(self, item):
    for pos in _hashes(item, self._k, self._m):
        if self._counters[pos] < self._max_val:
            self._counters[pos] += 1
```

When a counter reaches 15, subsequent additions that hash to that position are silently dropped. The counter no longer reflects the true number of items mapping to that slot.

## Why Saturation Breaks Deletion

The `remove` method (lines 117–124) reveals the consequence:

```python
def remove(self, item):
    positions = _hashes(item, self._k, self._m)
    for pos in positions:
        if self._counters[pos] == 0:
            raise ValueError("Item was not added to the filter")
    for pos in positions:
        if self._counters[pos] < self._max_val:  # saturated counters stay
            self._counters[pos] -= 1
```

The comment at line 124 — `# saturated counters stay` — is the key design decision. When a counter has saturated at 15, removal **refuses to decrement it**. This is intentional: if 20 items hash to the same position but the counter only reads 15, decrementing on removal would eventually drop the counter below the true count, creating **false negatives** (reporting an item absent when it's present). False negatives violate the fundamental Bloom filter invariant.

So saturated counters become permanent — they can never be decremented, which means:

1. **Items sharing a saturated slot can never be fully deleted.** The counter stays at 15 forever.
2. **The false positive rate degrades over time** as more positions become permanently set.
3. **Memory is wasted** — the standard counting Bloom filter uses 4× the space of a plain Bloom filter (4 bits per position vs. 1 bit), and saturated positions give back none of that investment.

## The Space Cost

The implementation actually uses a full byte per counter (`bytearray(self._m)` at line 110) for simplicity, even though only 4 bits are logically needed. A production counting Bloom filter packs two 4-bit counters per byte, but that's still 4× the memory of the standard `BloomFilter` which uses one bit per position (`bytearray((self._m + 7) // 8)` at line 30). For the same false positive rate, the counting variant needs roughly 4× the storage — and in this implementation, 8×.

## Why Cuckoo Filters Are the Better Answer

There is no cuckoo filter implementation in this codebase (the grep for `cuckoo|fingerprint|relocat` returned no relevant matches). This is a gap worth filling, because cuckoo filters solve exactly the problems visible here:

| Property | Counting Bloom Filter (this code) | Cuckoo Filter |
|---|---|---|
| Deletion | Partial — saturated counters are permanent | Perfect — fingerprint removal is exact |
| Space per item | ~4 bits/position × (m/n) positions | ~12 bits total per item |
| False positive rate at same space | Higher (counter overhead) | Lower (fingerprint precision) |
| Saturation risk | Yes — positions above 15 collisions break | None — no counters to overflow |

A cuckoo filter stores a small fingerprint of each item in a hash table with two candidate bucket positions. To delete, you find the fingerprint and remove it — no counter arithmetic, no saturation ceiling, no permanent false positives. The relocation ("cuckoo") step during insertion handles collisions by evicting existing fingerprints to their alternate bucket, giving high occupancy (~95%) without the pathological behavior this code exhibits at high load.

## The Tradeoff the Tests Don't Cover

The test suite (`bloom-filter/test_bloom_filter.py`) validates basic add/remove at lines 65–80 and double-add/remove at lines 88–95, but never exercises the saturation boundary. There is no test that adds 16+ items hashing to the same counter position and verifies that removal behaves correctly (or fails gracefully) under saturation. This is the most dangerous edge case in the implementation and it's untested.

## Topics to Explore

- [general] `cuckoo-filter-implementation` — Implement a cuckoo filter alongside the existing Bloom filter to directly compare space efficiency, deletion correctness, and false positive rates under high load
- [function] `bloom-filter/bloom_filter.py:remove` — Write a test that forces counter saturation (16+ collisions at one position) and verifies that removal refuses to create false negatives
- [general] `counter-width-tradeoffs` — Explore what happens with 8-bit or variable-width counters: the probability of needing more than 4 bits follows a Poisson distribution tied to k × n/m
- [file] `bloom-filter/bloom_filter.py` — The `_counters` array uses a full byte per position (line 110) but only 4 bits logically; a packed nibble representation would halve memory usage
- [general] `quotient-filters` — Another deletion-supporting alternative that uses quotienting rather than fingerprints, with different locality and cache performance characteristics

## Beliefs

- `counting-bloom-saturated-counters-are-permanent` — Once a CountingBloomFilter counter reaches `_max_val` (default 15), it is never decremented by `remove`, making items at that position permanently non-deletable
- `counting-bloom-4bit-default` — CountingBloomFilter defaults to 4-bit counters (`counter_bits=4`), giving a maximum count of 15 per position before saturation
- `counting-bloom-byte-per-counter-waste` — The implementation allocates one full byte per counter (`bytearray(self._m)`) despite using only 4 bits logically, doubling the actual memory footprint compared to packed nibbles
- `no-cuckoo-filter-in-codebase` — The repository contains no cuckoo filter implementation; the only deletion-supporting probabilistic filter is the counting Bloom filter
- `saturation-boundary-untested` — No test in `test_bloom_filter.py` exercises the counter saturation boundary (16+ collisions at a single position) to verify deletion correctness under overflow

