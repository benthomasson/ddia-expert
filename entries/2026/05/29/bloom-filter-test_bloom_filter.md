# File: bloom-filter/test_bloom_filter.py

**Date:** 2026-05-29
**Time:** 10:15

I'll work from the test file and repository structure since the implementation file isn't accessible.

## `bloom-filter/test_bloom_filter.py`

### Purpose

This is the test suite for a Bloom filter module that implements three variants: `BloomFilter`, `CountingBloomFilter`, and `ScalableBloomFilter`. It validates the probabilistic data structure's core guarantees — no false negatives, bounded false positives — along with serialization, set operations, and cardinality estimation. Within the DDIA reference implementations repo, it serves as the executable specification for Chapter 3's discussion of Bloom filters as used in LSM-tree storage engines (e.g., checking whether an SSTable might contain a key before doing disk I/O).

### Key Components

**Test groups by concern:**

| Test | What it validates |
|------|-------------------|
| `test_add_and_contains` | Basic insert + membership, `__len__` tracking |
| `test_no_false_negatives` | The fundamental Bloom filter invariant at scale (1000 items) |
| `test_false_positive_rate` | Empirical FPR stays below 2× the configured target (50k probes) |
| `test_optimal_parameters` | `bit_count` and `hash_count` match the textbook formulas: m = ⌈-n·ln(p) / ln²(2)⌉, k = round((m/n)·ln(2)) |
| `test_explicit_parameters` | Alternate constructor path: raw `bit_size` + `num_hashes` instead of probabilistic sizing |
| `test_counting_bloom_filter` | `CountingBloomFilter` supports `add`, `remove`, membership, and length |
| `test_counting_remove_nonexistent` | Removing an absent element raises `ValueError` |
| `test_counting_double_add_remove` | Reference-counting semantics: two adds require two removes |
| `test_serialization` | `to_bytes()` / `from_bytes()` round-trip preserves bit array, parameters, and membership |
| `test_union` | Bitwise OR merge of two compatible filters; `test_union_incompatible` rejects mismatched sizes |
| `test_estimate_count` | Cardinality estimation within 10% of actual, using the bit-density formula |
| `test_empty_filter` | Zero-item edge case: density 0, estimated FPR 0, count 0 |
| `test_duplicate_add` | Duplicates increment `len()` — the filter counts insertions, not distinct items |
| `test_determinism` | Same inputs produce identical `_bits` arrays — the hash function is deterministic, not randomized |
| `test_scalable_bloom_filter` | `ScalableBloomFilter` auto-grows by adding slices when capacity is exceeded |
| `test_bit_array_density` / `test_estimated_fpr` | Statistics accessors return sane values |

### Patterns

1. **Two constructor paths.** `BloomFilter` accepts either `(expected_items, false_positive_rate)` for auto-sizing or `(bit_size, num_hashes)` for explicit control. Tests cover both.

2. **Empirical probabilistic testing.** `test_false_positive_rate` doesn't just check the formula — it runs 50,000 non-member probes and measures the actual FPR. The 2× tolerance accounts for statistical variance while catching broken hash functions.

3. **Pythonic container protocol.** The implementation supports `in` (via `__contains__`), `len()` (via `__len__`), and direct attribute access (`_bits`), making Bloom filters feel like native Python collections.

4. **Variant taxonomy.** Three classes form a progression:
   - `BloomFilter` — fixed-size, no deletion
   - `CountingBloomFilter` — adds deletion via per-slot counters
   - `ScalableBloomFilter` — auto-grows by chaining sub-filters (`_slices`)

### Dependencies

**Imports:**
- `math` — for verifying the optimal-parameter formulas (ln, ceil)
- `random`, `string` — imported but unused in the current test suite (likely leftover from an earlier version or intended for fuzz-style tests)
- `pytest` — test framework, used only for `pytest.raises`
- `bloom_filter` — the module under test (`BloomFilter`, `CountingBloomFilter`, `ScalableBloomFilter`)

**Imported by:**
- Nothing imports this file directly. It's executed by `pytest`. The sibling `tester_test_bloom_filter.py` likely wraps or validates these tests as part of the project's meta-testing pattern (seen across all modules in this repo).

### Flow

Tests follow a consistent pattern:

1. **Construct** a filter with known parameters
2. **Insert** a set of items (usually string-formatted: `f"prefix-{i}"`)
3. **Assert** membership, non-membership, or statistical properties
4. For counting/scalable variants: exercise the additional API (remove, auto-growth)

The false-positive-rate test is the most involved: it inserts 5,000 members, then probes 50,000 distinct non-members, counts hits, and asserts the ratio is below the threshold.

### Invariants

1. **No false negatives** — every inserted item must be found. `test_no_false_negatives` checks this at scale; `test_add_and_contains` checks it at small scale.
2. **FPR ≤ 2× target** — the empirical false positive rate must not exceed double the configured rate.
3. **Optimal sizing follows textbook formulas** — `m` and `k` are derived from `n` and `p` using the standard formulas, not approximations.
4. **Counting filter is reference-counted** — `add("x")` twice then `remove("x")` once leaves `"x"` present.
5. **Union requires compatible parameters** — filters with different `bit_size` cannot be unioned.
6. **Cardinality estimation within 10%** — `estimate_count()` uses the bit-density-based estimator and must be reasonably accurate.
7. **Deterministic hashing** — identical inputs to identically-configured filters produce identical bit arrays.

### Error Handling

- **`ValueError` on invalid removal**: `CountingBloomFilter.remove()` raises `ValueError` when removing an element that was never added (tested by `test_counting_remove_nonexistent`).
- **`ValueError` on incompatible union**: `union()` raises `ValueError` when bit sizes or hash counts differ (tested by `test_union_incompatible`).
- No error handling for type mismatches on input — the filter presumably hashes whatever it receives.

## Topics to Explore

- [file] `bloom-filter/bloom_filter.py` — The implementation: how `_hashes` generates k hash positions, how the counting variant tracks per-slot counts, and how `ScalableBloomFilter` chains sub-filters with tightening FPR targets
- [function] `bloom-filter/bloom_filter.py:BloomFilter.estimate_count` — The cardinality estimator likely uses the formula n* = -(m/k)·ln(1 - X/m) where X is set bits; worth understanding its accuracy bounds
- [file] `bloom-filter/tester_test_bloom_filter.py` — The meta-test wrapper; this repo has a `tester_test_*.py` pattern across all modules that likely validates test quality or coverage
- [file] `log-structured-merge-tree/lsm.py` — LSM trees are the primary consumer of Bloom filters in DDIA; see how the filter integrates with SSTable lookups to avoid unnecessary disk reads
- [general] `scalable-bloom-filter-slice-growth` — How `ScalableBloomFilter` tightens the FPR for each new slice (typically geometric: p, p·r, p·r², ...) to keep the aggregate FPR bounded

## Beliefs

- `bloom-filter-no-false-negatives` — `BloomFilter.__contains__` never returns False for an item that was previously `add()`-ed; the test suite verifies this for up to 1000 items
- `bloom-filter-len-counts-insertions` — `BloomFilter.__len__` counts total `add()` calls, not distinct items — adding "dup" twice yields `len() == 2`
- `counting-bloom-filter-reference-counted` — `CountingBloomFilter` uses reference-counting semantics: an item added N times requires N `remove()` calls before it tests as absent
- `bloom-filter-optimal-sizing-textbook` — `BloomFilter` computes `bit_count` as ⌈-n·ln(p)/ln²(2)⌉ and `hash_count` as round((m/n)·ln(2)), matching the standard formulas from Bloom (1970)
- `bloom-filter-deterministic-hashing` — The hash function is deterministic (not seeded with randomness), so two filters with identical parameters produce identical bit arrays for identical inputs

