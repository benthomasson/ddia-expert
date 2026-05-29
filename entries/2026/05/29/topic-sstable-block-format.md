# Topic: How `block_size` affects the sparse index and binary search within blocks — the default of 4 in tests is suspiciously small and worth understanding

**Date:** 2026-05-29
**Time:** 08:57

The observations cut off mid-function at line 200. I have enough from the observations to explain the core mechanism, but I'll note what's missing.

---

# How `block_size` Controls the Sparse Index and Block Scanning

## The Core Mechanism

`block_size` determines how many entries sit between consecutive sparse index entries. It's the granularity knob for the classic **sparse index** pattern from DDIA Chapter 3: you don't index every key, you index every Nth key, then do a linear scan within the block.

### Writing: How the Index Gets Built

In `SSTableWriter.add` (`sstable.py:62`):

```python
if self._count % self._block_size == 0:
    self._index_entries.append((key_bytes, offset))
```

Every `block_size`th entry records its key and file offset into the sparse index. With `block_size=4`, you get an index entry at positions 0, 4, 8, 12, ... — so for 5 entries (apple, banana, cherry, date, elderberry), the index contains:
- Position 0 → `apple` (offset X)
- Position 4 → `elderberry` (offset Y)

That's just 2 index entries for 5 data entries.

With the production default of `block_size=64` (`sstable.py:46`), you'd need 64 entries before a second index entry appears. For 5 entries, you'd get a single index entry pointing to `apple` — and every lookup would scan from the start.

### Reading: Binary Search + Linear Scan

In `SSTableReader.get` (`sstable.py:192-200+`), the lookup is two-phase:

1. **Binary search the sparse index** to find which block the key might be in. The code at line 196 does a manual binary search over `self._index` (the list of `(key, offset)` pairs loaded from the file footer).

2. **Linear scan within the block** — once you know the block, you read entries sequentially from that offset until you either find the key, pass it lexicographically, or hit the next block boundary.

The `lsm.py` implementation makes this even clearer. At `lsm.py:119-131`:

```python
keys = [k for k, _ in self._sparse_index]
idx = bisect.bisect_right(keys, key) - 1
...
start_off = self._sparse_index[idx][1]
end_off = (self._sparse_index[idx + 1][1]
           if idx + 1 < len(self._sparse_index)
           else self._footer_start())
return self._scan_range_for_key(key, start_off, end_off)
```

`bisect_right` finds the rightmost index entry ≤ the target key. The scan range is bounded by the next index entry's offset (or the footer if it's the last block). The `_scan_range_for_key` method (`lsm.py:133-143`) then does a sequential read through that byte range.

## Why Tests Use `block_size=4`

The test file (`test_sstable.py`) uses `block_size=4` everywhere — every SSTableWriter call. This is deliberate:

1. **It forces multiple index entries with small datasets.** With 5 entries and `block_size=4`, you get 2 index entries. This actually exercises the binary search — the lookup has to choose between blocks. With the default of 64, every test dataset would have exactly 1 index entry and the binary search would be trivially degenerate (always pick block 0).

2. **It tests block boundary behavior.** Entry 4 (elderberry) starts a new block. A lookup for "date" (entry 3) must land in block 0, scan past apple/banana/cherry, and find date. A lookup for "elderberry" should land in block 1. A lookup for "fig" should land in block 1, scan elderberry, and report not-found.

3. **It's a test-only override.** The production default (`block_size=64` at `sstable.py:46`) trades a larger index for less scanning. The test value of 4 isn't meant to model production — it's the minimum size that exercises the multi-block code path without requiring hundreds of test entries.

## The Space-Time Tradeoff

| `block_size` | Index entries (for N keys) | Index memory | Worst-case scan per lookup |
|---|---|---|---|
| 4 (tests) | N/4 | Large | 4 entries |
| 16 (`lsm.py` default) | N/16 | Medium | 16 entries |
| 64 (`sstable.py` default) | N/64 | Small | 64 entries |

Smaller `block_size` = more index entries = more memory for the index but fewer entries to scan per lookup. The binary search step is O(log(N/block_size)), and the scan step is O(block_size). Total lookup cost: **O(log(N/B) + B)** where B is block_size.

## Two Implementations, Same Pattern

The codebase has two SSTable implementations with slightly different naming:

- **`lsm.py`**: calls it `sparse_index_interval` (default 16), uses Python's `bisect` module
- **`sstable.py`**: calls it `block_size` (default 64), implements binary search manually

Both follow the identical pattern: modulo check during writes, binary search + linear scan during reads. The `sstable.py` version is more production-oriented (has a proper file header/footer format with magic bytes and version), while `lsm.py` is simpler.

## What's Missing From These Observations

The `SSTableReader.get` code in `sstable.py` was cut off at line 200, right at the start of the binary search loop (`while lo <= hi`). The full block-scanning logic after the binary search is not visible. The `lsm.py` equivalent is complete and shows the same pattern, so the behavior is clear — but if you want to verify the `sstable.py` implementation handles edge cases correctly (e.g., key before first index entry, key after last index entry), you'd need to read past line 200.

---

## Topics to Explore

- [function] `sstable-and-compaction/sstable.py:SSTableReader.get` — The binary search + block scan was cut off at line 200; read the full implementation to see how it handles boundary conditions and the transition from index lookup to linear scan
- [function] `log-structured-merge-tree/lsm.py:_scan_range_for_key` — The linear scan within a block, including the early-exit optimization when the current key exceeds the target
- [function] `sstable-and-compaction/sstable.py:SSTableReader.range_scan` — Range scans use the same sparse index for the starting position but have different termination logic; worth comparing with point lookups
- [general] `index-size-vs-scan-cost-benchmarking` — Run the `test_large_dataset` test with different `block_size` values (4, 16, 64, 256) and measure actual lookup latency to see where the O(log(N/B) + B) tradeoff inflects
- [file] `log-structured-merge-tree/lsm.py` — Compare how `LSMTree` wires `sparse_index_interval` through to SSTable construction (line 203-207) and whether the interval is configurable per-level or global

## Beliefs

- `block-size-controls-index-density` — `block_size` (or `sparse_index_interval`) determines that every Nth entry is recorded in the sparse index; the index contains exactly ceil(entry_count / block_size) entries
- `sstable-lookup-is-two-phase` — Point lookups in both SSTable implementations first binary-search the sparse index to identify a block, then linearly scan entries within that block's byte range
- `test-block-size-differs-from-default` — Tests universally use `block_size=4` while the `SSTableWriter` default is 64; this ensures tests exercise multi-block index paths with small datasets
- `two-sstable-implementations-same-pattern` — `lsm.py` (using `sparse_index_interval=16` and `bisect`) and `sstable.py` (using `block_size=64` and manual binary search) implement the same sparse-index-with-block-scan pattern with different defaults and naming

