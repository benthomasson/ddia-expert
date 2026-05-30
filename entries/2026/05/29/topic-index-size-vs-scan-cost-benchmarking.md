# Topic: Run the `test_large_dataset` test with different `block_size` values (4, 16, 64, 256) and measure actual lookup latency to see where the O(log(N/B) + B) tradeoff inflects

**Date:** 2026-05-29
**Time:** 13:18

## Neither `test_large_dataset` nor `block_size` Exist in This Codebase

The observations confirm this: both grep searches returned zero matches. This investigation as literally stated **cannot be run** — it needs to be adapted to the actual API.

### What the codebase actually offers

The B-tree's branching factor is controlled by `max_keys_per_page`, not `block_size`. You can see this in the test constructors throughout `b-tree-storage-engine/test_btree.py` — every test uses `BTree(tmpdir, max_keys_per_page=4)` (e.g., lines 9, 58, 80). The closest existing test is `test_large` (line 57), which inserts 1,000 keys with `max_keys_per_page=4` and verifies all lookups.

The `PageManager` class (`b-tree-storage-engine/btree.py:28`) also accepts a `page_size` parameter (default 4096), which caps how many keys physically fit on a page regardless of `max_keys_per_page`.

### Why the O(log(N/B) + B) model doesn't quite apply here

The import at `btree.py:6` tells the real story:

```python
from bisect import bisect_left, bisect_right
```

This implementation uses **binary search within nodes**, not linear scan. That makes the actual lookup cost **O(log(N/B) · log(B))**, not O(log(N/B) + B). The additive B term in the classical model assumes sequential scan through a node's keys — `bisect_left` eliminates that.

This means the "inflection point" the investigation is looking for will be **much less dramatic** than the textbook predicts. With bisect, larger B almost always wins (shorter tree, fewer page reads) because the in-node cost grows logarithmically rather than linearly. The tradeoff only reasserts itself when B gets large enough that serialization/deserialization cost dominates — which depends on `page_size` and key/value sizes, not just `max_keys_per_page`.

### What you'd actually need to do

To run this investigation, you'd write a new test that:

1. Parameterizes `max_keys_per_page` over `[4, 16, 64, 256]`
2. Inserts a large dataset (10K+ keys) for each value
3. Calls `pm.reset_counters()` (`btree.py:112`) before each lookup, then reads `pm.pages_read` after — this gives you I/O cost directly
4. Times lookups with `time.perf_counter()` for wall-clock latency

But you'd also need to increase `page_size` proportionally for large `max_keys_per_page` values (256 keys won't fit in a 4096-byte page with any reasonable key/value sizes), which introduces a confound: larger pages mean more data per I/O operation.

### The deeper issue

The `test_page_io_counts` test (line 95) already validates the core invariant: lookups read at most `height + 1` pages. Since height = ceil(log_B(N)), increasing B reduces height and thus page reads. The interesting experiment isn't the lookup inflection — it's the **insert throughput** tradeoff, because larger nodes mean more data to serialize and rewrite on every split (`_serialize_leaf` at line 192, `_serialize_internal` which would be further in the file).

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_search` — The actual lookup traversal logic; confirm it uses bisect and count page reads per level
- [function] `b-tree-storage-engine/btree.py:_serialize_leaf` — Serialization cost grows linearly with keys-per-page, which is where the real large-B penalty lives
- [general] `page-size-vs-max-keys-interaction` — How `page_size` caps effective `max_keys_per_page` and whether the code enforces or silently truncates
- [function] `b-tree-storage-engine/btree.py:PageManager.reset_counters` — The instrumentation hook for measuring I/O; any benchmark should use this
- [general] `split-cost-at-high-branching-factor` — Insert splits rewrite entire nodes; at B=256, each split serializes 128+ key-value pairs, making insert latency the real tradeoff

## Beliefs

- `btree-uses-bisect-not-linear-scan` — In-node key lookup uses `bisect_left`/`bisect_right`, making within-node search O(log B) rather than the textbook O(B)
- `no-block-size-parameter-exists` — The branching factor is controlled by `max_keys_per_page`, not `block_size`; no `test_large_dataset` test exists
- `page-io-counters-are-resettable` — `PageManager` tracks `pages_read` and `pages_written` with a `reset_counters()` method, enabling per-operation I/O measurement
- `page-size-constrains-max-keys` — `page_size` (default 4096) physically limits how many keys fit per node; `max_keys_per_page=256` would require a proportionally larger `page_size` to avoid `ValueError` on oversized entries

