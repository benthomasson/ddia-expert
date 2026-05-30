# Topic: Production systems split merge output into multiple fixed-size SSTables rather than one giant file; this implementation merges into a single file, which limits LCS correctness (L1+ should have non-overlapping key ranges across files)

**Date:** 2026-05-29
**Time:** 13:25

I'll work with the observations provided, which give us enough to explain the key issue. Let me also check what's in the LSM tree module's compaction logic.

---

## Single-File Merge Output vs. Production-Style Multi-File Splitting

### The core problem

In Leveled Compaction Strategy (LCS), each level beyond L0 must maintain a critical invariant: **no two SSTables at the same level can have overlapping key ranges**. This is what makes leveled compaction efficient for reads — you only need to check one SSTable per level for any given key.

Production systems like LevelDB and RocksDB enforce this by splitting merge output into multiple fixed-size SSTables (typically 64MB or similar). When you merge an L0 SSTable with overlapping L1 files, the output gets cut into pieces that each cover a non-overlapping slice of the key space:

```
L0:  [a---------z]         (one file, full range)
L1:  [a--d] [e--k] [l--z]  (three files, non-overlapping)

After merge → L1:  [a--d] [e--k] [l--z]  (still three files, same size target)
```

### What this implementation does

Looking at `sstable-and-compaction/sstable.py`, the `merge_sstables` function (called at line 43 of `test_sstable.py`) takes a list of readers and a **single output path**:

```python
merged_path = os.path.join(tmpdir, 'merged.sst')
merged = merge_sstables([reader2, reader], merged_path, remove_tombstones=True)
```

The function signature reveals the constraint: it writes everything into one file at `merged_path`. There's no `max_file_size` parameter, no callback for splitting, no mechanism to produce multiple output files from a single merge operation.

The `CompactionManager` (tested at lines 49–55 of `test_sstable.py`) uses this same merge path. When it runs leveled compaction (lines 101–116), it merges L0 files into L1 — but the result is a single SSTable:

```python
result = mgr.run_compaction()
assert len(result) >= 1
assert result[0].level == 1
```

### Why this breaks LCS correctness

Consider what happens after several rounds of compaction. At L1 you'd accumulate files like:

```
L1: [a--z]          ← first compaction produced one big file
L1: [a--m]          ← second compaction overlaps with the first!
```

These two L1 files have overlapping key ranges (`a` through `m` appears in both). This violates the fundamental LCS invariant. In a correct implementation, the second compaction would:

1. Select the L1 file `[a--z]` because it overlaps with the incoming data
2. Merge them together
3. **Split the output** into, say, `[a--f]`, `[g--m]`, `[n--z]` — each under the target size

Without splitting, you get one of two bad outcomes:
- **Unbounded file growth**: L1 becomes a single enormous file, defeating the purpose of leveled compaction (bounded read amplification)
- **Overlapping ranges**: Multiple L1 files cover the same keys, so reads must check all of them — you've regressed to size-tiered behavior

### The SSTableWriter's role

The `SSTableWriter` class (`sstable.py:42–96`) is designed for writing a single file. It opens one file handle, builds one sparse index, and writes one header/footer pair. There's no `split_at(size_threshold)` method. To fix this, you'd need either:

1. A wrapper that monitors bytes written and rotates to a new `SSTableWriter` when a threshold is hit
2. A modified merge function that yields boundary points and creates multiple writers

### What's correct in the implementation

The merge logic itself — using a heap to pick the entry with the smallest key, resolving timestamp conflicts, optionally dropping tombstones — is sound. The multi-way merge test (lines 89–100 of `test_sstable.py`) correctly verifies that when multiple SSTables contain the key `'shared'`, the highest timestamp wins (`val_4`). The merge algorithm is right; it's the I/O boundary (one file vs. many) that's simplified.

---

## Topics to Explore

- [function] `sstable-and-compaction/sstable.py:merge_sstables` — The merge function itself: examine how the heap-based k-way merge works, how it resolves duplicate keys by timestamp, and where you'd inject a size-based file rotation
- [function] `sstable-and-compaction/sstable.py:CompactionManager.run_compaction` — See how files are selected for compaction at each level and how the single-file merge output gets assigned to L1 — this is where the split logic would need to live
- [general] `leveldb-table-splitting` — LevelDB's `DoCompactionWork` splits output at 2MB boundaries and also when the output file starts overlapping too many grandparent (L+2) files — understanding this policy explains why naive single-file merge causes cascading problems at deeper levels
- [file] `log-structured-merge-tree/lsm.py` — The LSM tree module has its own SSTable and compaction implementation; compare whether it has the same single-file limitation or handles splitting differently
- [general] `size-tiered-vs-leveled-write-amplification` — Understanding why LCS accepts higher write amplification (rewriting files to maintain non-overlapping ranges) in exchange for bounded read amplification clarifies why the split-on-output step is non-negotiable for correctness

## Beliefs

- `merge-produces-single-file` — `merge_sstables` accepts one output path and writes all merged entries into a single SSTable file, with no size-based splitting
- `lcs-l1-can-have-overlapping-ranges` — The `CompactionManager` with `strategy='leveled'` does not enforce non-overlapping key ranges across L1+ files because merge output is never split
- `merge-resolves-duplicates-by-timestamp` — When multiple SSTables contain the same key, the multi-way merge keeps only the entry with the highest timestamp (verified by the `'shared'` key test at line 97 of `test_sstable.py`)
- `sstable-writer-single-file-lifecycle` — `SSTableWriter` binds to one file descriptor at construction and has no mechanism to rotate to a new file mid-write, making it structurally incompatible with size-bounded output splitting
- `compaction-result-assigned-level-1` — `CompactionManager.run_compaction` with leveled strategy assigns merged output to level 1 (asserted at line 115 of `test_sstable.py`), but does not verify that the new file's key range is disjoint from existing L1 files

