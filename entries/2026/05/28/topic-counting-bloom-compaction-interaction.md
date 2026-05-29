# Topic: How CountingBloomFilter's `remove()` interacts with SSTable compaction (merge old tables → rebuild or update filter)

**Date:** 2026-05-28
**Time:** 19:10

# CountingBloomFilter's `remove()` and SSTable Compaction: The Missing Integration

## Short Answer

**They don't interact — yet.** The bloom filter module (`bloom-filter/bloom_filter.py`) and the SSTable/compaction modules (`sstable-and-compaction/sstable.py`, `log-structured-merge-tree/lsm.py`) are independent DDIA concept demonstrations with zero cross-references. Grepping for "bloom" across both LSM and SSTable directories returns no matches. But the `CountingBloomFilter` was clearly *designed* with compaction in mind — understanding why requires tracing both sides.

## What CountingBloomFilter Provides

The `CountingBloomFilter` class (`bloom-filter/bloom_filter.py:104–140`) extends the standard `BloomFilter` by replacing single bits with **saturating 4-bit counters**. When you `add()` a key, it increments `k` counter positions (using the same Kirschner-Mitzenmacher double hashing at lines 8–12). When you `remove()` a key, it decrements them.

This is the critical capability: a standard bloom filter is add-only. Once a bit is set, you can never unset it because multiple keys may share that bit position. Counters solve this — decrementing a counter doesn't destroy information about other keys that also hash to that position (unless the counter has saturated, which the 4-bit ceiling handles conservatively).

## What Compaction Does to Keys

The codebase has two compaction implementations with fundamentally different scopes:

### Full compaction (`log-structured-merge-tree/lsm.py:compact`, ~line 319)

Merges **every** SSTable into one. Collects all entries, sorts by `(key, -seq)`, deduplicates by keeping the newest version, and **permanently drops tombstones**. After this, the old SSTables are deleted from disk. There's only one surviving SSTable.

### Partial compaction (`sstable-and-compaction/sstable.py:CompactionManager`)

Uses STCS or LCS to merge **subsets** of SSTables. The `merge_sstables()` function (line ~251) does a k-way heap merge with `(key, -timestamp)` ordering — newest-wins dedup, but tombstones are preserved (`remove_tombstones` defaults to `False` and no call site overrides it).

In both cases, compaction **drops keys** — either because a newer version supersedes them, or because a tombstone marks them as deleted. These dropped keys are the ones a bloom filter would need to un-learn.

## The Design Tension: Rebuild vs. Incremental Update

When compaction produces a new SSTable from old inputs, its bloom filter could be constructed in two ways:

### Option A: Rebuild from scratch (standard BloomFilter)

After `merge_sstables()` writes the output, scan the output to build a fresh `BloomFilter` from the surviving keys. This is simple and avoids any counter-overflow issues. The cost is O(N) where N is the number of keys in the output — but you already paid O(N) to write the SSTable, so this adds constant-factor overhead, not algorithmic overhead.

This is what production systems (LevelDB, RocksDB) actually do. They build the bloom filter as part of the table-building process — `SSTableWriter.add()` would insert each key into a fresh filter, and `SSTableWriter.finish()` would serialize it into the footer alongside the sparse index.

### Option B: Incremental update (CountingBloomFilter)

Take the old SSTable's `CountingBloomFilter`, call `remove()` for each dropped key, and `add()` for each newly merged-in key. This avoids rebuilding from scratch.

This is where `CountingBloomFilter.remove()` would theoretically shine — but it has practical problems:

1. **You need to know which keys were dropped.** The current `merge_sstables()` silently skips duplicate and tombstoned keys (lines ~280–284 in the merge loop). It would need to be instrumented to report what it discarded.

2. **Merging N inputs into 1 output means N filters → 1 filter.** You can't incrementally patch N separate filters into one — the counter positions are specific to each filter's `m` (bit array size). You'd need to union them first, but `CountingBloomFilter` doesn't support union (only the standard `BloomFilter` has `union()` at lines ~80–90).

3. **Counter saturation.** The 4-bit counters saturate at 15. After saturation, `remove()` becomes a no-op for that position to avoid false negatives. Over many compaction cycles, saturation accumulates and the filter degrades.

## Why Rebuild Wins in Practice

For SSTable compaction, **rebuild is almost always the right choice**:

- Compaction already reads every key (O(N) I/O cost dwarfs filter construction)
- A fresh filter has zero saturation — optimal false positive rate
- No need to track which keys were dropped
- Simpler code with fewer failure modes

`CountingBloomFilter.remove()` is more useful for **in-memory data structures** where you want to maintain a filter without rescanning the backing data — for example, removing a key from a memtable's filter before flushing. For SSTable compaction, where you're rewriting the entire file anyway, the rebuild approach is strictly better.

## What Integration Would Look Like

If the modules were connected, the natural integration points would be:

1. **`SSTableWriter.add()` (`sstable-and-compaction/sstable.py`)** — Insert each key into a `BloomFilter` (not counting) as entries are written
2. **`SSTableWriter.finish()`** — Serialize the filter via `to_bytes()` into the SSTable footer, alongside the sparse index (the 12-byte header format at `bloom_filter.py:92–101` is designed for this)
3. **`SSTableReader.__init__()`** — Deserialize the filter from the footer via `from_bytes()`
4. **`SSTableReader.get()`** — Check `key not in self._bloom` before the sparse index binary search — this is where the I/O savings happen
5. **`merge_sstables()`** — No special bloom filter logic needed; the output writer builds a fresh filter naturally through step 1

The `CountingBloomFilter` would remain relevant for the memtable layer if the LSM tree supported key deletion from the active memtable, but the current implementation marks deletions with tombstone entries rather than removing memtable keys.

---

## Topics to Explore

- [function] `bloom-filter/bloom_filter.py:CountingBloomFilter` — Read the `remove()` implementation to understand saturating counter semantics and the 4-bit ceiling tradeoff
- [function] `sstable-and-compaction/sstable.py:SSTableWriter.finish` — The footer-writing logic where bloom filter bytes would be appended during SSTable construction; compare with `BloomFilter.to_bytes` format
- [function] `sstable-and-compaction/sstable.py:CompactionManager._lcs_compact` — Leveled compaction selects overlapping SSTables for merge; understanding this selection logic reveals why per-SSTable bloom filters are more valuable here than in full compaction
- [general] `memtable-bloom-filter-lifecycle` — The one scenario where `CountingBloomFilter.remove()` is genuinely useful: maintaining a filter over a mutable in-memory table where keys can be overwritten or deleted without a full rescan
- [file] `sstable-and-compaction/test_sstable.py` — Test scenarios for merge and compaction that would need extension to cover bloom filter construction and false-positive-rate guarantees post-compaction

## Beliefs

- `bloom-compaction-rebuild-not-patch` — For SSTable compaction, rebuilding a fresh BloomFilter during the merge write pass is simpler and more correct than incrementally patching a CountingBloomFilter with `remove()`; compaction already pays O(N) I/O so filter construction adds only constant-factor overhead
- `counting-bloom-remove-needs-drop-tracking` — Using `CountingBloomFilter.remove()` during compaction would require `merge_sstables` to report which keys were discarded, which the current implementation does not do (duplicates and tombstones are silently skipped in the merge loop)
- `bloom-filter-not-integrated` — Neither LSM tree implementation (`lsm.py` or `sstable.py`) references or uses the bloom filter module; they exist as independent DDIA concept demonstrations with zero cross-module imports
- `counting-bloom-useful-for-memtable` — `CountingBloomFilter.remove()` is architecturally suited for maintaining filters over mutable in-memory structures (memtables), not for SSTable compaction where the entire file is rewritten
- `sstable-writer-is-natural-filter-builder` — The `SSTableWriter.add()` → `finish()` lifecycle is the natural place to construct a per-SSTable bloom filter, since keys are already being iterated in sorted order during the write pass

