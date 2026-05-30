# Function: BloomFilter in bloom-filter/bloom_filter.py

**Date:** 2026-05-29
**Time:** 10:14

## `BloomFilter` — Space-Efficient Probabilistic Set Membership

### Purpose

`BloomFilter` is a classic probabilistic data structure that answers "is this item in the set?" with two possible outcomes: **definitely not** or **probably yes**. It trades a controlled false positive rate for dramatic space savings over exact set representations like hash sets. This is the foundational building block in the file — `CountingBloomFilter` and `ScalableBloomFilter` both build on top of it.

In DDIA's context, Bloom filters appear in LSM-tree storage engines (like LevelDB/RocksDB) to avoid unnecessary disk reads: before checking an SSTable on disk, the engine queries its Bloom filter to skip SSTables that definitely don't contain the key.

### Contract

**Preconditions:**
- Items passed to `add` and `__contains__` must be strings (the `_hashes` function calls `.encode("utf-8")`)
- Either provide both `bit_size` and `num_hashes` (manual configuration) or neither (auto-sizing from `expected_items` and `false_positive_rate`)

**Postconditions:**
- After `add(x)`, `x in filter` is always `True` (no false negatives)
- `x in filter` may return `True` for items never added (false positive), with probability bounded by the configured rate *when item count stays at or below `expected_items`*

**Invariants:**
- `self._bits` is always `ceil(m / 8)` bytes long
- Bits are only ever set, never cleared — the filter is monotonic
- `self._count` tracks the number of `add()` calls, not the number of distinct items

### Parameters

| Parameter | Type | Default | Meaning |
|-----------|------|---------|---------|
| `expected_items` | int | 1000 | Anticipated number of distinct items. Used to size the bit array. |
| `false_positive_rate` | float | 0.01 | Target FPR (1%). Smaller values require more bits. |
| `bit_size` | int or None | None | Override: set `m` directly. Must be paired with `num_hashes`. |
| `num_hashes` | int or None | None | Override: set `k` directly. Must be paired with `bit_size`. |

**Edge cases:** If only one of `bit_size`/`num_hashes` is provided, the other is silently ignored and the auto-sizing path runs — this is likely unintentional and not enforced.

### Return Values

- `add()` returns `None`; mutates the filter in place.
- `__contains__()` returns `bool` — `False` is definitive, `True` is probabilistic.
- `estimate_count()` returns `float` — can be `0.0`, `inf`, or an estimate using the inverse of the bit-filling formula.
- `union()` returns a new `BloomFilter` instance; does not mutate either operand.
- `to_bytes()` returns `bytes` — a 12-byte header (`m`, `k`, `count` as little-endian uint32s) followed by the raw bit array.

### Algorithm

**Auto-sizing (`__init__`):**

The optimal bit array size `m` and hash count `k` for a target false positive rate `p` with `n` expected items are:

```
m = ceil(-n * ln(p) / (ln2)²)
k = round((m / n) * ln2)
```

These come from minimizing the FPR formula `(1 - e^(-kn/m))^k`. The implementation computes these directly, clamping both to a minimum of 1.

**Hashing (`_hashes`):**

Uses Kirschner-Mitzenmacker double hashing: a single MD5 digest is split into two 64-bit values `h1` and `h2`, then `k` positions are generated as `(h1 + i * h2) % m`. This is a well-known technique that provides pairwise-independent hash functions from a single hash computation.

**Membership test (`__contains__`):**

Checks all `k` bit positions. Short-circuits on the first unset bit (definite negative). Returns `True` only if all `k` positions are set.

**Union:**

Requires identical `m` and `k`. Performs bitwise OR of the underlying byte arrays. The resulting `_count` is the sum of both counts, which is an overcount if items overlap — this is a known limitation since the filter can't determine overlap.

**Cardinality estimation (`estimate_count`):**

Uses the formula `-(m/k) * ln(1 - X/m)` where `X` is the number of set bits. This inverts the expected bit density formula to estimate how many distinct items were inserted. Returns `inf` when all bits are set (the log term would be undefined).

### Side Effects

- `add()` mutates `self._bits` and increments `self._count`
- All other methods are pure reads
- No I/O, no threading concerns, no global state

### Error Handling

- `union()` raises `ValueError` if the two filters have different `m` or `k`
- `from_bytes()` will raise `struct.error` if data is shorter than 12 bytes
- No validation that string items are passed — non-strings will fail at `_hashes` with `AttributeError` on `.encode()`
- No overflow protection on `_count` (Python ints don't overflow, but the `struct.pack("<I", ...)` in `to_bytes` will fail if count exceeds 2³² - 1)

### Usage Patterns

```python
# Typical: auto-sized for expected workload
bf = BloomFilter(expected_items=10_000, false_positive_rate=0.001)
bf.add("key-123")
if "key-456" in bf:
    # probably present — do the expensive lookup
    pass

# Merging filters from different SSTable levels
merged = bf1.union(bf2)

# Serialization for persistence alongside SSTable files
data = bf.to_bytes()
restored = BloomFilter.from_bytes(data)
```

Callers must understand that `len(bf)` returns the number of `add()` calls, not distinct items. For distinct-item estimation, use `estimate_count()`.

### Dependencies

- `hashlib.md5` — hash function (via `_hashes`)
- `math` — `log`, `ceil` for sizing formulas
- `struct` — binary serialization (little-endian uint32 packing)

No external dependencies. The `_hashes` module-level function is tightly coupled — it's the only hash strategy and isn't pluggable.

### Assumptions Not Enforced

1. Items are assumed to be strings — no type check before calling `_hashes`
2. Providing only one of `bit_size`/`num_hashes` silently falls through to auto-sizing
3. `union()` sums `_count` which overcounts when filters share items
4. `estimated_false_positive_rate()` uses `density^k` which is an approximation (it assumes bit positions are independent, which becomes less true as the filter fills)
5. MD5 is used for hashing — fine for data structure purposes but not cryptographic; the choice prioritizes speed and uniform distribution

## Topics to Explore

- [function] `bloom-filter/bloom_filter.py:_hashes` — The double-hashing scheme that maps items to k bit positions using a single MD5 digest
- [function] `bloom-filter/bloom_filter.py:ScalableBloomFilter` — How the filter grows automatically by chaining slices with geometrically tightening FPR targets
- [function] `bloom-filter/bloom_filter.py:CountingBloomFilter` — How saturating counters enable deletion at the cost of extra space
- [file] `bloom-filter/test_bloom_filter.py` — Test cases that verify FPR bounds, union semantics, and serialization round-trips
- [general] `bloom-filter-in-lsm-trees` — How storage engines like LevelDB/RocksDB attach Bloom filters to SSTables to skip unnecessary disk reads during point lookups

## Beliefs

- `bloom-filter-no-false-negatives` — After `add(x)`, `__contains__(x)` always returns True; the filter never produces false negatives
- `bloom-filter-double-hashing` — Hash positions are generated via Kirschner-Mitzenmacker double hashing (h1 + i*h2) % m from a single MD5 digest, not k independent hash functions
- `bloom-filter-monotonic-bits` — Bits in the filter are only ever set, never cleared; this makes the structure append-only and union-compatible
- `bloom-filter-count-is-insertions` — `_count` tracks the number of `add()` calls, not distinct items; `estimate_count()` uses bit density for cardinality estimation instead
- `bloom-filter-union-requires-identical-params` — `union()` requires both filters to have the same m and k; it raises ValueError otherwise, since bitwise OR is only meaningful when bit positions map to the same hash space

