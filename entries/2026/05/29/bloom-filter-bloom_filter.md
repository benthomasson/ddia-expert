# File: bloom-filter/bloom_filter.py

**Date:** 2026-05-29
**Time:** 13:26

# `bloom-filter/bloom_filter.py`

## Purpose

This file implements three variants of the Bloom filter data structure, a core concept from Chapter 3 of *Designing Data-Intensive Applications*. Bloom filters are used in storage engines (LSM-trees, SSTables) to avoid expensive disk reads for keys that don't exist. The file provides a standard Bloom filter, a counting variant that supports deletion, and a scalable variant that grows automatically — covering the progression from textbook to production-grade.

## Key Components

### `_hashes(item, k, m)` — Hash Position Generator

Uses **double hashing** with MD5 to produce `k` positions in `[0, m)`. The MD5 digest is split into two 8-byte halves interpreted as little-endian integers `h1` and `h2`, then positions are computed as `(h1 + i * h2) % m`. This is the Kirschner-Mitzenmacher optimization — two hash functions simulate `k` independent ones. All three filter classes share this single function.

### `BloomFilter` — Standard Probabilistic Set

The constructor accepts either semantic parameters (`expected_items`, `false_positive_rate`) or explicit parameters (`bit_size`, `num_hashes`). When using semantic parameters, it derives optimal sizing:

- `m = ceil(-n * ln(p) / ln(2)²)` — bit array size
- `k = round((m/n) * ln(2))` — number of hash functions

The bit array is stored as a `bytearray` of `ceil(m/8)` bytes, with manual bit manipulation (`pos // 8` for byte index, `pos % 8` for bit offset). This is more memory-efficient than a list of booleans.

Notable methods beyond add/contains:

- **`estimate_count()`** — Uses the inverse of the bit-filling formula: `-(m/k) * ln(1 - x/m)`. This estimates the number of *distinct* items from bit density alone, useful when `_count` isn't reliable (e.g., after `union`).
- **`estimated_false_positive_rate()`** — Computes the actual FPR from current bit density as `density^k`, which diverges from the target FPR as the filter fills.
- **`union(other)`** — Bitwise OR of two filters. Only valid when `m` and `k` match. The resulting `_count` is the sum, which overstates distinct items if there's overlap.
- **`to_bytes()` / `from_bytes()`** — Binary serialization with a 12-byte header (`m`, `k`, `count` as unsigned 32-bit LE integers) followed by the raw bit array. This is the format you'd use to persist a filter alongside an SSTable.

### `CountingBloomFilter` — Deletion-Supporting Variant

Replaces the bit array with a `bytearray` of per-position counters (one byte each, but logically bounded to `counter_bits` — default 4 bits, max value 15). This enables `remove()` by decrementing counters.

Key design choice: **counter saturation**. When a counter hits `_max_val`, `add()` stops incrementing it, and `remove()` won't decrement it either. This prevents counter overflow from corrupting the filter at the cost of potential false positives from permanently-set positions.

The `remove()` method does a two-pass check: first verifies all positions are non-zero (raising `ValueError` if not), then decrements. This avoids partial decrements on items that weren't added.

### `ScalableBloomFilter` — Auto-Growing Filter

Maintains a list of `BloomFilter` slices. Each new slice gets geometrically larger capacity (`initial_cap * growth^idx`) and a tighter FPR target (`p * ratio^idx`). The tightening FPR ensures the aggregate false positive rate across all slices stays bounded — the geometric series `p * (r^0 + r^1 + ...)` converges to `p / (1 - r)`.

Membership checks scan all slices (`any()`), so lookup cost grows linearly with the number of slices. This is the tradeoff for unbounded growth without rehashing.

## Patterns

- **Double hashing** — A single MD5 call produces all `k` hash positions, avoiding `k` independent hash computations. This is the standard approach from Bloom filter literature.
- **Bytearray as bit vector** — Manual bit packing avoids the overhead of Python's `bitarray` or boolean lists while keeping the implementation dependency-free.
- **Dual constructor** — `BloomFilter` accepts either capacity-based or explicit sizing via `bit_size`/`num_hashes`. The explicit path is used internally by `union()` and `from_bytes()` to construct filters with known parameters.
- **Geometric scaling** — The scalable variant uses exponential growth (capacity) with exponential decay (FPR), a pattern borrowed from Almeida et al.'s scalable Bloom filter paper.

## Dependencies

**Imports:** Only stdlib — `hashlib` (MD5 for hashing), `math` (log/ceil for sizing), `struct` (binary serialization). No external dependencies.

**Imported by:** `test_bloom_filter.py` and `tester_test_bloom_filter.py` — the test suite and its validation harness.

## Flow

**Add path:** `item` (string) → `_hashes()` encodes to UTF-8, hashes with MD5, splits digest into `h1`/`h2`, generates `k` positions → each position sets a bit via `|=` → `_count` increments.

**Lookup path:** Same hash computation → each position checked via `&` → short-circuits to `False` on any zero bit, returns `True` only if all `k` bits are set.

**Serialization round-trip:** `to_bytes()` packs header + bit array → `from_bytes()` unpacks header, constructs filter with explicit sizing, overwrites `_bits` and `_count`.

**Scalable growth:** `add()` checks `len(current) >= cap` → if full, `_add_slice()` creates a new `BloomFilter` with doubled capacity and halved FPR target → item goes into the new slice.

## Invariants

1. **`m` and `k` are always ≥ 1** — enforced by `max(1, ...)` in the constructor.
2. **Bit array size matches `m`** — `bytearray((m + 7) // 8)` allocates exactly enough bytes.
3. **Union requires identical `m` and `k`** — `ValueError` if mismatched, since bitwise OR is only meaningful for identically-parameterized filters.
4. **Counters saturate, never overflow** — `CountingBloomFilter.add()` checks `< _max_val` before incrementing.
5. **Remove is atomic or fails** — `CountingBloomFilter.remove()` checks all positions before decrementing any.
6. **Scalable FPR tightens geometrically** — each slice's target FPR is `p * ratio^idx`, ensuring the union of all slices doesn't exceed the overall target.

## Error Handling

Minimal and deliberate:

- **`CountingBloomFilter.remove()`** raises `ValueError` if any counter is zero — the only explicit error in the module. This is a precondition violation (removing something not present).
- **`BloomFilter.union()`** raises `ValueError` on parameter mismatch.
- **`estimate_count()`** returns `float("inf")` when all bits are set rather than raising — a sentinel value indicating the filter is saturated.
- No input validation on `item` type — `_hashes()` will raise `AttributeError` if `item` lacks `.encode()`, relying on Python's duck typing.

## Topics to Explore

- [file] `bloom-filter/test_bloom_filter.py` — How the FPR guarantees and edge cases (saturation, serialization round-trips, scalable growth) are verified
- [function] `bloom-filter/bloom_filter.py:_hashes` — The double-hashing scheme is the core correctness assumption; worth verifying the distribution quality and collision behavior
- [general] `bloom-filter-in-lsm-trees` — How LSM-tree implementations (see `log-structured-merge-tree/lsm.py`) use Bloom filters to skip disk reads during lookups
- [function] `bloom-filter/bloom_filter.py:remove` — The two-pass remove and counter saturation interact in subtle ways; saturated counters cause silent false-positive leaks after removal
- [general] `scalable-bloom-filter-fpr-proof` — Why the geometric FPR decay (`p * ratio^i`) keeps the aggregate rate bounded; the Almeida et al. paper formalizes this

## Beliefs

- `bloom-filter-double-hashing` — All three filter classes use double hashing from a single MD5 digest, generating k positions as `(h1 + i*h2) % m` rather than computing k independent hashes
- `bloom-filter-counting-saturation-leak` — Saturated counters in `CountingBloomFilter` are never decremented by `remove()`, causing permanent false positives for any item whose hash positions overlap with a saturated counter
- `bloom-filter-scalable-fpr-geometric` — `ScalableBloomFilter` assigns each slice a FPR of `p * ratio^idx`, so the aggregate FPR converges to `p / (1 - ratio)` — with default `ratio=0.5`, the aggregate target is `2p`
- `bloom-filter-union-count-overestimates` — `BloomFilter.union()` sets `_count = self._count + other._count`, which overstates distinct items when filters share elements; `estimate_count()` from bit density is more accurate in that case
- `bloom-filter-remove-is-two-pass` — `CountingBloomFilter.remove()` validates all counters are non-zero before decrementing any, preventing partial state corruption on items not in the filter

