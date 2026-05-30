# Function: _hashes in bloom-filter/bloom_filter.py

**Date:** 2026-05-29
**Time:** 10:13

## `_hashes(item, k, m)` — Double Hashing for Bloom Filters

### Purpose

`_hashes` generates `k` distinct bit positions within a bit array of size `m` for a given item. Bloom filters need multiple independent hash functions to map each item to several positions; this function produces all `k` positions from a single MD5 digest using the **double hashing** technique, avoiding the cost and complexity of maintaining `k` separate hash functions.

### Contract

- **Precondition**: `item` must be a string (calls `.encode("utf-8")`). `k` and `m` must be positive integers.
- **Postcondition**: Returns exactly `k` integers, each in `[0, m)`.
- **Invariant**: The same `(item, k, m)` triple always produces the same list of positions (deterministic, no randomness).

### Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `item` | `str` | The value to hash. Converted to UTF-8 bytes before hashing. |
| `k` | `int` | Number of hash positions to generate (the bloom filter's hash count). |
| `m` | `int` | Size of the bit array (the bloom filter's bit count). Positions are taken modulo `m`. |

**Edge cases**: If `k=0`, returns an empty list. If `m=1`, every position is `0`. The function does not guard against `m=0` (would raise `ZeroDivisionError`).

### Return Value

A list of `k` integers, each in `[0, m)`. Positions are **not guaranteed unique** — two different values of `i` can produce the same `(h1 + i * h2) % m`, especially when `m` is small relative to `k`. Callers (like `BloomFilter.add`) are idempotent with respect to duplicate positions (setting a bit twice is harmless), so this is safe in practice.

### Algorithm

1. **Hash the item**: Compute the 16-byte (128-bit) MD5 digest of the UTF-8–encoded string.
2. **Split into two 64-bit integers**:
   - `h1` = first 8 bytes, interpreted as a little-endian unsigned integer.
   - `h2` = last 8 bytes, interpreted as a little-endian unsigned integer.
3. **Generate positions via double hashing**: For each `i` in `[0, k)`, compute `(h1 + i * h2) % m`. This is the Kirsch–Mitzenmacher optimization — the linear combination of two hash values simulates `k` independent hash functions with negligible loss in false-positive rate.

The key insight is that `h1` acts as the base position and `h2` acts as the step size. Each successive hash index advances by another `h2` (mod `m`), producing a deterministic sequence of positions spread across the bit array.

### Side Effects

None. Pure function — no mutation, no I/O, no state changes.

### Error Handling

- Raises `AttributeError` if `item` is not a string (no `.encode`).
- Raises `ZeroDivisionError` if `m=0` (unguarded modulo).
- MD5 computation on `hashlib.md5(...)` will not raise for valid byte inputs.

No exceptions are caught — all errors propagate to the caller.

### Usage Patterns

Called by every operation that needs to map an item to bit positions:

```python
# BloomFilter.add — set bits
for pos in _hashes(item, self._k, self._m):
    self._bits[pos // 8] |= 1 << (pos % 8)

# BloomFilter.__contains__ — check bits
for pos in _hashes(item, self._k, self._m):
    if not (self._bits[pos // 8] & (1 << (pos % 8))):
        return False

# CountingBloomFilter uses it identically for counter positions
```

Callers are responsible for ensuring `item` is a string and that `k`/`m` match the filter's parameters.

### Dependencies

- **`hashlib`** (stdlib): MD5 digest computation. MD5 is used for uniform distribution, not cryptographic security — this is appropriate since bloom filters need good bit dispersion, not collision resistance.

### Assumptions Not Enforced by Types

1. **`item` is a string** — no type annotation or runtime check. Passing bytes, integers, or None will fail at `.encode()`.
2. **`m > 0`** — division by zero is not guarded.
3. **MD5 produces sufficient uniformity** — the double-hashing scheme assumes `h1` and `h2` behave as independent uniform random variables. MD5's 128-bit output split into two 64-bit halves satisfies this in practice, but the code doesn't verify or enforce it.
4. **UTF-8 encoding is injective for the input domain** — two distinct strings that encode to the same bytes would collide, but this can't happen with valid Python strings since UTF-8 encoding is a bijection.

---

## Topics to Explore

- [general] `kirsch-mitzenmacher-double-hashing` — The 2006 paper proving that double hashing preserves bloom filter false-positive guarantees with only two hash functions, which is the theoretical foundation for this implementation
- [function] `bloom-filter/bloom_filter.py:BloomFilter.__init__` — How `m` and `k` are derived from the target false-positive rate and expected item count using the optimal bloom filter formulas
- [function] `bloom-filter/bloom_filter.py:CountingBloomFilter.remove` — How counter saturation interacts with the positions returned by `_hashes` — saturated counters are never decremented, which can cause silent false negatives after removal
- [file] `bloom-filter/test_bloom_filter.py` — Test coverage for hash distribution, collision behavior, and edge cases around small `m` or `k` values
- [general] `md5-vs-murmur-for-bloom-filters` — Why production bloom filters (like those in LevelDB/RocksDB) typically use murmur3 instead of MD5 — faster, and the double-hashing scheme doesn't need cryptographic properties

---

## Beliefs

- `bloom-double-hashing-two-from-one` — `_hashes` derives `k` hash positions from a single MD5 call using Kirsch–Mitzenmacher double hashing: `(h1 + i * h2) % m` for `i` in `[0, k)`
- `bloom-hashes-pure-deterministic` — `_hashes` is a pure function with no side effects; identical inputs always produce identical output positions
- `bloom-hashes-positions-not-unique` — The `k` positions returned by `_hashes` are not guaranteed to be distinct — duplicate positions can occur, and all callers are written to tolerate this
- `bloom-md5-split-64bit` — The 128-bit MD5 digest is split at the midpoint into two 64-bit little-endian integers used as the two base hash values
- `bloom-hashes-no-m-zero-guard` — `_hashes` does not guard against `m=0`, which would cause a `ZeroDivisionError` in the modulo operation

