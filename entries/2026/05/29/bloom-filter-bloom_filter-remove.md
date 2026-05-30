# Function: remove in bloom-filter/bloom_filter.py

**Date:** 2026-05-29
**Time:** 10:18



## `CountingBloomFilter.remove`

### Purpose

This method removes an item from a counting Bloom filter by decrementing the counters at the item's hash positions. Standard Bloom filters can't support deletion because clearing a bit might affect other items that hash to the same position. Counting Bloom filters solve this by replacing single bits with counters — `remove` is the operation that makes that tradeoff worthwhile.

### Contract

- **Precondition**: The item must have been previously added — specifically, all counter positions for the item must be non-zero. This is a necessary but not sufficient condition (false positives mean a never-added item could pass this check).
- **Postcondition**: Each non-saturated counter at the item's hash positions is decremented by 1. `self._count` decreases by 1.
- **Invariant**: Saturated counters (those at `self._max_val`) are never decremented, because the filter can't know whether the true count is exactly `_max_val` or higher.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `self` | `CountingBloomFilter` | The filter instance |
| `item` | `str` | The item to remove. Must be a string (passed to `_hashes`, which calls `.encode("utf-8")`) |

**Edge cases**: If `item` was never added but all its hash positions happen to be non-zero due to collisions from other items, the removal will silently succeed — a false positive removal. This is inherent to probabilistic data structures.

### Return Value

Returns `None`. The effect is entirely through mutation.

### Algorithm

The method works in two passes over the hash positions:

1. **Compute positions**: Call `_hashes(item, self._k, self._m)` to get `k` bucket indices via double-hashing on MD5.

2. **Validation pass** (lines 3–5): Check every position. If *any* counter is zero, the item definitely wasn't added — raise `ValueError`. This is done as a separate pass to avoid partial decrements: if position 2 of 5 is zero, you don't want to have already decremented positions 0 and 1.

3. **Decrement pass** (lines 6–8): For each position, decrement the counter *only if* it hasn't saturated to `_max_val`. A saturated counter is left untouched because the true count could be `_max_val + n` — decrementing would undercount.

4. **Update count**: Decrement `self._count` by 1.

The two-pass design is the key detail — it provides atomicity-like behavior within a single-threaded context: either all counters are decremented, or none are.

### Side Effects

- Mutates `self._counters` (decrements up to `k` entries)
- Mutates `self._count` (decrements by 1)

### Error Handling

- **`ValueError`**: Raised if any hash position has a zero counter, indicating the item is definitely not present. Critically, this check happens *before* any mutation, so the filter state is unchanged on error.
- No protection against removing an item more times than it was added (beyond the zero-counter check). If you add "foo" once and two other items happen to share all of "foo"'s hash positions, you could remove "foo" even after those other items are removed — the counters would still be positive from the collisions.

### Usage Patterns

```python
cbf = CountingBloomFilter(expected_items=100, false_positive_rate=0.01)
cbf.add("alice")
cbf.add("bob")

cbf.remove("alice")       # succeeds, decrements counters
assert "alice" not in cbf  # likely true, but not guaranteed (other items may hold those counters up)

cbf.remove("carol")       # raises ValueError — carol was never added (probably)
```

Callers should be aware that `remove` on a counting Bloom filter can introduce **false negatives** — something standard Bloom filters never have. If items A and B collide on a position, removing A might drop that counter to zero, making B appear absent.

### Dependencies

- **`_hashes(item, k, m)`**: The shared double-hashing function using MD5. Same function used by `add` and `__contains__`, which is essential for correctness — all three must agree on positions.
- **`self._counters`**: A `bytearray` with one byte per counter (not bit-packed), so counters can go up to 255 but are capped at `_max_val` (default `(1 << 4) - 1 = 15`).
- **`self._max_val`**: Derived from `counter_bits` at construction. Controls the saturation ceiling.

## Topics to Explore

- [function] `bloom-filter/bloom_filter.py:_hashes` — The double-hashing scheme that maps items to positions; understanding collision behavior is critical to understanding false positive removals
- [general] `counter-saturation-probability` — How likely counters are to saturate at 4-bit width, and the implications for remove correctness
- [function] `bloom-filter/bloom_filter.py:CountingBloomFilter.add` — The dual of `remove`; compare the saturation-clamping logic in both directions
- [file] `bloom-filter/test_bloom_filter.py` — Test cases that exercise the add/remove cycle, false positive removal, and the ValueError path
- [general] `cuckoo-filters` — An alternative to counting Bloom filters that supports deletion with better space efficiency and without the saturated-counter problem

## Beliefs

- `cbf-remove-two-pass-atomicity` — `remove` validates all hash positions are non-zero before decrementing any, preventing partial state corruption on failure
- `cbf-saturated-counters-never-decremented` — Counters that reached `_max_val` during `add` are permanently frozen and skipped by `remove`, accepting potential false negatives to avoid underflow
- `cbf-remove-false-positive-acceptance` — `remove` can succeed for items never added if hash collisions from other items keep all positions non-zero — this is a fundamental property, not a bug
- `cbf-remove-can-introduce-false-negatives` — Unlike standard Bloom filters, counting Bloom filter deletion can cause false negatives when colliding items share counter positions

