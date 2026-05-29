# Function: CountingBloomFilter in bloom-filter/bloom_filter.py

**Date:** 2026-05-29
**Time:** 07:34

# `CountingBloomFilter`

## Purpose

A standard Bloom filter is append-only — once a bit is set, you can never unset it because multiple items may share that bit position. `CountingBloomFilter` solves this by replacing each single bit with a multi-bit counter, enabling **deletion** of items while preserving the probabilistic membership guarantee. Each hash position tracks how many items map to it, so decrementing on removal is safe as long as the counter hasn't saturated.

This is the classic extension described in Fan et al. (2000) and referenced in DDIA's discussion of probabilistic data structures.

## Contract

**Preconditions:**
- Items passed to `add`, `remove`, and `__contains__` must be strings (they're passed to `_hashes`, which calls `.encode("utf-8")`).
- `remove` requires the item to have been previously added — specifically, all hash positions must have non-zero counters.

**Postconditions:**
- After `add(x)`, `x in filter` is guaranteed `True` (no false negatives).
- After `remove(x)`, `x in filter` may still be `True` if other items share all of `x`'s hash positions (false positive).
- `__len__` tracks the number of `add` calls minus `remove` calls, not distinct items.

**Invariants:**
- Counter values are in `[0, _max_val]`. Counters that reach `_max_val` are **saturated** and never increment or decrement further.
- `_m` and `_k` are always ≥ 1.

## Parameters

### `__init__`

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `expected_items` | int | 1000 | Anticipated number of distinct items. Used to size the counter array and choose the hash count. |
| `false_positive_rate` | float | 0.01 | Target FPR. Lower values → more counters (`_m`) and more hashes (`_k`). |
| `counter_bits` | int | 4 | Bits per counter, controlling the saturation ceiling (`2^counter_bits - 1`). With the default of 4, counters saturate at 15. |

**Edge cases:** If `expected_items` is 0 or `false_positive_rate` is 1.0, the formulas degenerate — `_m` could be 1 and `_k` could be 0 (clamped to 1). There's no validation guarding against nonsensical inputs like negative values or FPR ≥ 1.

## Return Value

- `__contains__` returns `bool` — `True` if all counter positions are non-zero (may be a false positive).
- `__len__` returns `int` — the net add/remove count, which can go negative if `remove` is called more times than `add` for a given item (the code doesn't prevent this once the first-pass zero-check passes).
- `add` and `remove` return `None`.

## Algorithm

### Sizing (`__init__`)

Uses the same optimal Bloom filter formulas as `BloomFilter`:

1. **Counter array size** `m = ceil(-n * ln(p) / (ln2)²)` — minimizes FPR for a given capacity.
2. **Hash count** `k = round((m / n) * ln2)` — the optimal number of hash functions.
3. **Max counter value** `2^counter_bits - 1` (default 15).
4. Allocates a `bytearray` of length `m` — one full byte per counter, even though only `counter_bits` bits are logically used. This trades ~2× space for simpler indexing.

### `add(item)`

1. Compute `k` hash positions via `_hashes`.
2. For each position, increment the counter **unless it's already at `_max_val`** (saturation guard).
3. Increment `_count`.

### `remove(item)`

Two-pass approach:

1. **Validation pass:** Compute all `k` positions. If any counter is 0, raise `ValueError` — the item was never added (or was already removed).
2. **Decrement pass:** For each position, decrement the counter **only if it hasn't saturated**. A saturated counter stays at `_max_val` because the true count is unknown once saturation occurred — decrementing would risk undercount.
3. Decrement `_count`.

### `__contains__(item)`

Returns `True` iff all `k` hash positions have counters > 0. Identical semantics to a standard Bloom filter's membership test, just reading counters instead of bits.

## Side Effects

- `add` and `remove` mutate `self._counters` and `self._count` in place.
- No I/O, no external calls.

## Error Handling

- `remove` raises `ValueError` if any hash position has a zero counter. This is the only explicit error path.
- **Not caught:** If `remove` is called for an item that wasn't added but whose hash positions overlap with other items' positions (all counters > 0), the removal silently succeeds and corrupts the filter. This is inherent to the data structure — membership is probabilistic, so removal correctness is also probabilistic.
- Counter saturation is handled silently — no warning when a counter hits `_max_val`. A saturated counter effectively becomes "sticky," preventing accurate removal for any item that hashes to that position.

## Usage Patterns

```python
cbf = CountingBloomFilter(expected_items=10000, false_positive_rate=0.001)

cbf.add("user:42")
assert "user:42" in cbf

cbf.remove("user:42")
# "user:42" in cbf is now *probably* False, but could still be True
```

**Caller obligations:**
- Only remove items you've previously added. The zero-check catches some violations but not all (due to hash collisions).
- If you expect high multiplicity (same item added many times), increase `counter_bits` to avoid saturation. With `counter_bits=4`, any position hit 15+ times is permanently stuck.
- The filter does **not** deduplicate — adding the same item twice increments counters twice, and you must remove it twice to fully undo.

## Dependencies

- **`_hashes(item, k, m)`** (module-level function): Double-hashing scheme using MD5. Produces `k` positions in `[0, m)` from a single MD5 digest via `h1 + i*h2 mod m`. This is the Kirby-Steele technique — two hash functions simulate `k` independent ones.
- **`hashlib`**: MD5 digest in `_hashes`.
- **`math`**: `log` and `ceil` for parameter sizing.

### Unspoken Assumptions

1. **Items are strings.** The `_hashes` function calls `.encode("utf-8")` — passing bytes, ints, or other types will fail at runtime with no type guard.
2. **MD5 is sufficient for uniformity.** MD5 is not cryptographically secure, but that's irrelevant here — what matters is uniform distribution, which MD5 provides adequately.
3. **One byte per counter is acceptable.** Even with `counter_bits=4`, each counter occupies a full byte. For large `m`, this is 8× the memory of a packed representation.
4. **`_count` can go negative.** There's no guard against removing items that were never added when hash collisions mask the absence. The `_count` field is a simple add/remove tally, not a true cardinality estimate.
5. **No thread safety.** Concurrent `add`/`remove` calls can race on counter reads and writes.

---

## Topics to Explore

- [function] `bloom-filter/bloom_filter.py:_hashes` — The double-hashing scheme that all three filter variants share; understanding its collision properties is key to reasoning about FPR and saturation risk
- [function] `bloom-filter/bloom_filter.py:BloomFilter` — The standard (non-counting) variant; comparing the two clarifies exactly what counters buy you and what they cost in space
- [file] `bloom-filter/test_bloom_filter.py` — Test cases reveal the expected usage patterns and edge cases the author considered, especially around removal semantics
- [general] `counter-saturation-probability` — For a given workload, how likely is it that a 4-bit counter saturates? The answer determines whether `counter_bits=4` is safe or needs to be higher
- [general] `cuckoo-filters` — The modern alternative to counting Bloom filters that supports deletion with better space efficiency; worth comparing trade-offs

## Beliefs

- `cbf-saturated-counters-never-decrement` — When a counter reaches `_max_val`, it is permanently frozen: neither `add` nor `remove` will change it, making removal of any item hashing to that position a no-op
- `cbf-remove-false-accept` — `remove` can silently succeed for an item that was never added if all its hash positions have non-zero counters from other items, corrupting the filter state
- `cbf-one-byte-per-counter` — Each counter uses a full byte regardless of `counter_bits`, so the filter uses 8× more memory than a bit-packed representation would
- `cbf-count-tracks-calls-not-cardinality` — `__len__` returns `add` calls minus `remove` calls, not distinct element count; adding the same item twice and removing once leaves `len == 1` even though the item is still present

