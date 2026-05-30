# Topic: RocksDB extends LevelDB's merge with a `MergeOperator` that supports partial merges during compaction — the next step beyond what both implementations show

**Date:** 2026-05-29
**Time:** 10:13

# RocksDB's MergeOperator: Beyond Last-Writer-Wins Compaction

## What These Implementations Do

Both LSM implementations in this repo follow a strict **last-writer-wins** strategy during compaction. Look at how `lsm.py` handles merging during compaction:

At `log-structured-merge-tree/lsm.py:275-298`, the `compact()` method collects entries from all SSTables and the memtable into a `merged` dict, always keeping only the entry with the highest priority (newest sequence number):

```python
merged: dict = {}  # key -> (priority, value_bytes)
# ...
merged[k] = (priority, v)  # newest wins, older values discarded
```

The SSTable module takes the same approach. In `sstable-and-compaction/sstable.py`, `merge_sstables` (called at `test_sstable.py:41`) merges multiple readers, picks the entry with the highest timestamp when keys collide, and optionally strips tombstones. The test at line 44-45 confirms: `apple` becomes `green` (timestamp 2.0) and `banana` disappears (tombstoned at 2.0).

Both implementations treat values as **opaque blobs to be replaced**. During compaction, older values for the same key are simply thrown away.

## What's Missing: The Read-Modify-Write Problem

Consider a counter that gets incremented thousands of times, or a set where elements are appended. Under last-writer-wins, every update must:

1. **Read** the current value (potentially scanning memtable + multiple SSTables)
2. **Modify** it in application code
3. **Write** the full new value back

This is expensive. Every increment of a counter requires a full read path just to write `counter + 1`. With high write rates, this becomes a bottleneck — the read amplification defeats the purpose of an LSM tree's write-optimized design.

## How RocksDB's MergeOperator Solves This

RocksDB introduces a `MergeOperator` interface that lets you define how values **combine** rather than replace. Instead of `Put(key, new_value)`, you call `Merge(key, operand)` — writing a delta without reading the existing value.

The operator has two methods:

- **`FullMerge(existing_value, operands)`** — called during reads or compaction when a base `Put` value is found. Combines the base value with all stacked merge operands.
- **`PartialMerge(operand1, operand2)`** — called during compaction when **no base value is present**. Collapses two operands into one without needing the full value.

### Why Partial Merges Matter During Compaction

This is the critical insight. In these implementations, compaction at `lsm.py:319-347` walks all SSTables and produces a single output. If RocksDB encounters a chain of `Merge` records for a key during compaction but the base `Put` is in a deeper level that isn't being compacted, it can't do a `FullMerge`. Instead, it calls `PartialMerge` to collapse adjacent operands — reducing the chain length without needing the base value at all.

For example, a counter with 1000 pending `+1` operands gets collapsed to a single `+1000` operand during compaction, even if the base value lives in L3 and only L0-L1 are being compacted.

### What Would Change in This Code

To support merge operators, the compaction logic at `lsm.py:275-298` would need to stop treating values as opaque replacements. Instead of `merged[k] = (priority, v)`, it would need to:

1. Distinguish between `Put` entries and `Merge` entries (a new entry type alongside `TOMBSTONE`)
2. Accumulate merge operands per key during the k-way merge at `lsm.py:323`
3. Call `FullMerge` when a `Put` base is found, or `PartialMerge` when only operands exist
4. Write the collapsed result to the output SSTable at `lsm.py:342`

The SSTable format in `sstable-and-compaction/sstable.py` would also need a new entry type — currently entries are either values or tombstones (`TOMBSTONE_MARKER = 0xFF` at line 11), but merge operands are a third category that must survive compaction if no base value is available to resolve them against.

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:compact` — The k-way merge loop that would need to become merge-operator-aware; trace how it currently discards all but the newest value
- [function] `sstable-and-compaction/sstable.py:merge_sstables` — The multi-reader merge that picks winners by timestamp; compare with how RocksDB would accumulate operands here
- [general] `rocksdb-merge-operator-interface` — Read RocksDB's wiki on MergeOperator to see the full FullMerge/PartialMerge contract and the AssociativeMergeOperator shortcut for commutative operations
- [file] `sstable-and-compaction/sstable.py` — The entry format (lines 60-80) and tombstone marker design; extending this with a merge-operand type is the prerequisite for any MergeOperator support
- [general] `leveled-compaction-partial-merge` — How leveled compaction (tested at `test_sstable.py:105-119`) interacts with partial merges: L0→L1 compaction may only see operands while the base value sits in L2+

## Beliefs

- `lsm-compaction-last-writer-wins` — Both LSM implementations resolve key conflicts during compaction by keeping only the newest value; older values are unconditionally discarded with no user-defined merge logic
- `sstable-entry-types-binary` — The SSTable format supports exactly two entry types (value and tombstone via `TOMBSTONE_MARKER`); there is no third type for merge operands, which would be required for RocksDB-style merge support
- `compaction-requires-full-read-for-update` — Without a merge operator, any read-modify-write workload (counters, append sets) must perform a full read path before writing, negating the LSM tree's write amplification advantage
- `partial-merge-avoids-cross-level-reads` — RocksDB's `PartialMerge` enables compaction to collapse merge operands without accessing the base value in deeper levels, which is impossible under the current implementations' all-or-nothing merge at `lsm.py:275-298`

