# Topic: For a given workload, how likely is it that a 4-bit counter saturates? The answer determines whether `counter_bits=4` is safe or needs to be higher

**Date:** 2026-05-29
**Time:** 10:16

## 4-Bit Counter Saturation in CountingBloomFilter

### The mechanism

In `bloom-filter/bloom_filter.py:106`, a 4-bit counter has `_max_val = (1 << 4) - 1 = 15`. When `add()` is called (line 112–114), each of the `k` hash positions gets its counter incremented — but only up to 15. Once a counter hits 15, it **stays there permanently**: `remove()` at line 124 explicitly skips decrementing saturated counters (`if self._counters[pos] < self._max_val`). This is correct behavior — decrementing a saturated counter would undercount and cause false negatives, which are worse than false positives.

### The probability model

Each of the `m` counter positions receives hits from `n` items, each touching `k` positions. The number of hits on any single position follows approximately a **Poisson distribution** with:

$$\lambda = \frac{n \cdot k}{m}$$

At optimal Bloom filter parameters (which this implementation computes at lines 103–105), `k = (m/n) · ln2`, so:

$$\lambda = \frac{n \cdot k}{m} = \ln 2 \approx 0.693$$

With λ ≈ 0.693, the distribution is heavily concentrated at 0 and 1:

| Count | Probability |
|-------|------------|
| 0     | ~50.0%     |
| 1     | ~34.7%     |
| 2     | ~12.0%     |
| 3     | ~2.8%      |
| 4     | ~0.5%      |
| ≥15   | ~10⁻¹⁵     |

**At designed capacity, the probability of any single counter reaching 15 is astronomically small.** Even with `m ≈ 9,585` positions (for `n=1000, p=0.01`), the expected number of saturated counters is effectively zero.

### When it breaks: overloading the filter

The danger comes when you insert far more items than `expected_items`. If you insert 10× the designed capacity (10,000 items into a filter sized for 1,000), then λ jumps to ~7.3, and:

- P(counter ≥ 15) ≈ 0.005 per position
- Expected saturated counters ≈ 44 out of 9,585

At 20× overload, λ ≈ 14.6, and roughly **half** of all counters would saturate. The filter becomes functionally a standard (non-counting) Bloom filter — you can no longer remove items.

### Why saturation is insidious

The real problem isn't just false positives — it's that **removal silently stops working**. Look at `remove()` (lines 118–124): if the item's hash positions include a saturated counter, that counter stays at 15 after "removal." The item may still appear present, and so will any future item whose hashes overlap with that stuck counter. There's no error, no warning — the `_count` still decrements (line 125), so `len()` lies about the filter's state.

### Verdict

**4 bits is safe for normal operation** (inserting ≤ `expected_items` elements with balanced add/remove). It becomes unsafe when:
1. The filter is significantly overloaded (>5× designed capacity)
2. There's a hot-spot pattern where specific positions get disproportionate hits (unlikely with MD5 double-hashing, but possible with adversarial inputs)

The classic literature (Fan et al., 2000) found that 4 bits is sufficient for most practical workloads, and this implementation follows that convention. If you need to handle sustained overload or have paranoia about hash collisions, bump to `counter_bits=8` — the space cost is negligible since this implementation already uses a full byte per counter (line 107: `bytearray(self._m)`).

### The hidden irony

Look at line 107: `self._counters = bytearray(self._m)` allocates **8 bits per counter** regardless of `counter_bits`. The `counter_bits` parameter only controls the logical saturation point, not the actual memory layout. You're already paying for 8-bit counters in memory — you just aren't using the upper 4 bits. Setting `counter_bits=8` costs exactly zero additional memory in this implementation.

---

## Topics to Explore

- [function] `bloom-filter/bloom_filter.py:_hashes` — The double-hashing scheme determines how uniformly counters are loaded; non-uniform hashing would make saturation more likely at specific positions
- [general] `counting-bloom-packed-counters` — A production implementation would pack multiple 4-bit counters per byte to actually save space, unlike the current one-byte-per-counter layout
- [function] `bloom-filter/bloom_filter.py:remove` — Trace through a remove-after-saturation scenario to see how silent false positives emerge
- [general] `spectral-bloom-filters` — An extension where counter values encode approximate multiplicity, making the saturation/precision tradeoff more explicit
- [file] `bloom-filter/test_bloom_filter.py` — The test suite has no saturation test; adding one that overloads the filter and verifies remove behavior would catch the silent failure mode

---

## Beliefs

- `counting-bloom-4bit-safe-at-capacity` — At designed capacity (n ≤ expected_items), the expected number of saturated 4-bit counters is effectively zero because the per-position load follows Poisson(ln2 ≈ 0.693)
- `counting-bloom-saturation-breaks-remove` — When a counter saturates at `_max_val`, `remove()` silently skips decrementing it, causing permanent false positives with no error signal
- `counting-bloom-wasted-bits` — The `bytearray(self._m)` allocation at line 107 uses 8 bits per counter regardless of `counter_bits`, so increasing `counter_bits` up to 8 costs zero additional memory
- `counting-bloom-overload-threshold` — At 10× overload (~10× expected_items inserted), roughly 0.5% of counters saturate; at 20× overload, approximately half saturate

