# Topic: Neither implementation includes Bloom filters, which DDIA describes as critical for reducing the number of SSTables checked during point lookups, especially under size-tiered compaction

**Date:** 2026-05-29
**Time:** 08:59

# Missing Bloom Filters in SSTable/LSM Implementations

## The Problem: Unnecessary Disk I/O on Point Lookups

Both SSTable implementations use **sparse indexes** to locate keys, but neither integrates a Bloom filter to short-circuit lookups for keys that don't exist in a given SSTable. This means every point lookup must probe SSTables sequentially until the key is found or all tables are exhausted.

### How lookups work today

In `log-structured-merge-tree/lsm.py`, the `SSTable.get()` method (line ~119) performs a binary search on the sparse index, then scans forward through the data block:

```python
def get(self, key: str) -> Tuple[bool, Optional[bytes]]:
    if not self._sparse_index:
        return False, None
    keys = [k for k, _ in self._sparse_index]
    idx = bisect.bisect_right(keys, key) - 1
    ...
    return self._scan_range_for_key(key, start_off, end_off)
```

In `sstable-and-compaction/sstable.py`, `SSTableReader.get()` (line ~188) does the same — binary search on sparse index, then linear scan within a block. Neither checks a Bloom filter first.

The LSM tree's top-level `get()` iterates through **every SSTable** in reverse sequence order until it finds the key. If the key doesn't exist in *any* SSTable, it reads from *all* of them — each time opening the file, seeking to the sparse index region, and scanning a data block. That's O(N) disk reads where N is the number of SSTables.

### Why this matters more under size-tiered compaction

The `log-structured-merge-tree/lsm.py` implementation uses what is effectively size-tiered compaction (line ~316): it merges all SSTables into one when the count crosses a threshold. The `sstable-and-compaction/sstable.py` implementation explicitly supports both `size_tiered` and `leveled` strategies.

Under size-tiered compaction, SSTables at the same "tier" have **overlapping key ranges**. Unlike leveled compaction — where each level (except L0) has non-overlapping key ranges, letting you skip an entire level based on min/max key bounds — size-tiered compaction cannot use key-range metadata to rule out an SSTable. The key *could* be in any of them. This makes the Bloom filter's role critical: it's the only mechanism that can cheaply say "this key is definitely not here" without reading data blocks.

### What DDIA prescribes

DDIA describes attaching a Bloom filter to each SSTable. Before performing the sparse-index lookup and block scan, the storage engine checks the Bloom filter. If the filter says "not present," the SSTable is skipped entirely — no file I/O needed. With a well-tuned false positive rate (typically 1%), this eliminates nearly all unnecessary disk reads for missing keys.

### The irony

The repo already has a complete Bloom filter implementation in `bloom-filter/bloom_filter.py` (176 lines), including standard, counting, and scalable variants, plus serialization (`to_bytes`/`from_bytes` at lines 89–100). It's production-ready for embedding into SSTable metadata. But neither SSTable module imports or references it — confirmed by the grep results showing zero matches for `bloom` or `filter` in either implementation file.

Integrating it would mean:
1. Building a `BloomFilter` during `SSTable.write()` / `SSTableWriter.add()` as entries are written
2. Serializing it into the SSTable footer (or a sidecar file)
3. Loading it in `SSTable.__init__()` / `SSTableReader.__init__()`
4. Checking it as the first step in `get()`, returning "not found" immediately on a negative

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:LSMTree.get` — Trace the full read path to see how many SSTables are probed for a missing key
- [file] `bloom-filter/bloom_filter.py` — Understand the serialization API (`to_bytes`/`from_bytes`) that would plug into SSTable footers
- [function] `sstable-and-compaction/sstable.py:SSTableWriter.finish` — Where the Bloom filter bytes would be written alongside the sparse index in the footer
- [general] `size-tiered-vs-leveled-read-amplification` — Compare how overlapping key ranges in size-tiered compaction amplify the cost of missing-key lookups versus leveled compaction's non-overlapping guarantees
- [function] `sstable-and-compaction/sstable.py:CompactionManager.run_compaction` — How compaction rebuilds SSTables and where new Bloom filters would need to be generated for merged output

## Beliefs

- `lsm-get-probes-all-sstables-on-miss` — `LSMTree.get()` iterates through every SSTable in reverse order and returns only after checking all of them when a key is absent
- `sstable-sparse-index-no-bloom` — Both SSTable implementations use only sparse indexes for point lookups; no probabilistic filter guards against unnecessary disk reads
- `bloom-filter-exists-but-unused` — `bloom-filter/bloom_filter.py` provides a complete, serializable Bloom filter with configurable FPR, but neither SSTable module imports it
- `size-tiered-compaction-overlapping-ranges` — The LSM tree's compaction strategy produces SSTables with overlapping key ranges, making min/max key bounds insufficient to skip tables during lookups

