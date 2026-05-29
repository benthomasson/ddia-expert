# File: sstable-and-compaction/sstable.py

**Date:** 2026-05-28
**Time:** 18:23

# `sstable-and-compaction/sstable.py`

## Purpose

This file is a self-contained implementation of **Sorted String Tables (SSTables)** — the on-disk data structure at the heart of LSM-tree storage engines like LevelDB, RocksDB, and Cassandra. It owns three responsibilities:

1. **Binary file format** — writing and reading a custom SSTable format with header, sorted entries, sparse index, and footer.
2. **K-way merge** — combining multiple SSTables into one while deduplicating by key and resolving conflicts by timestamp.
3. **Compaction strategies** — both size-tiered (STCS) and leveled (LCS) compaction, implementing the two major strategies discussed in DDIA Chapter 3.

This corresponds directly to the "SSTables and LSM-Trees" section of DDIA, providing a concrete implementation of concepts that are typically described at whiteboard level.

## Key Components

### Constants and Wire Format

```
MAGIC = b"SSTB"          # 4-byte file identifier
HEADER_FMT = ">4sHI"     # magic(4) + version(2) + entry_count(4) = 10 bytes
FOOTER_FMT = ">QI"        # index_offset(8) + index_count(4) = 12 bytes
TOMBSTONE_MARKER = 0xFF   # single-byte sentinel for deleted keys
```

The file layout is: `[header][entries...][sparse index][footer]`. The footer is at the end so the reader can seek to EOF minus `FOOTER_SIZE` to bootstrap — a pattern shared with real SSTable formats.

### `SSTableEntry` / `SSTableMetadata`

Simple data carriers. `SSTableEntry.value = None` encodes a tombstone (delete marker). `SSTableMetadata` captures key range, entry count, level, and timestamp — everything needed for compaction decisions without opening the file.

### `SSTableWriter`

**Contract:** Caller must add keys in sorted order. The writer builds a sparse index by recording every `block_size`-th entry's key and byte offset.

- `add(key, value, timestamp)` — appends one entry to the file. Tombstones are encoded as a single `0xFF` byte instead of a value-length + value payload.
- `finish()` — writes the sparse index block, the footer, then seeks back to byte 0 to patch the header with the final entry count. Returns `SSTableMetadata`.

The entry encoding is: `[key_len:2][key_bytes][timestamp:8][tombstone_byte | value_len:4 + value_bytes]`. Note the asymmetry: tombstones are 1 byte, values are 4+N bytes. This means the `_read_entry` parser reads 1 byte and branches on whether it equals `0xFF`.

### `SSTableReader`

Reads an SSTable file, loading the sparse index into memory on construction.

- **`get(key)`** — binary searches the sparse index to find which block the key might be in, then scans entries within that block. This is O(log B + S) where B is the number of index entries and S is the block size.
- **`scan()`** — full table iteration in sorted order.
- **`range_scan(start_key, end_key)`** — filters `scan()` with `[start_key, end_key)` bounds. Notably does *not* use the sparse index to skip — it scans from the beginning.

On construction, the reader also determines `_min_key`, `_max_key`, and `_timestamp` by reading the first entry and scanning from the last index entry to find the final entry. These are cached for metadata queries.

### `merge_sstables(readers, output_path, remove_tombstones=False)`

A classic **k-way merge** using a min-heap. The heap entries are `(key, -timestamp, source_index, entry, iterator)`. The negative timestamp means that when two entries share a key, the newest one is popped first. All subsequent entries with the same key are skipped via the `last_key` guard — this is how last-write-wins conflict resolution works.

`remove_tombstones=True` drops delete markers, which is appropriate only at the bottom level of compaction where there's no older data that needs to be shadowed.

### `CompactionManager`

Manages a collection of SSTables and implements two compaction strategies:

**Size-Tiered (STCS):** Groups SSTables into buckets by file size tier. When any tier accumulates `min_threshold` (default 4) SSTables, they're merged into one. Simple and write-optimized — good for write-heavy workloads but can cause space amplification.

**Leveled (LCS):** Organizes SSTables into levels with exponentially increasing size caps (`level_base_size * fanout^(level-1)`). L0 compacts into L1 when it has `l0_trigger` SSTables. Higher levels compact by picking the SSTable with the most overlap with the next level and merging it down. This bounds space amplification and read amplification at the cost of more write amplification.

## Patterns

- **Append-then-patch header**: The writer writes a placeholder header, appends all data, then seeks back to patch the entry count. This avoids buffering or two-pass writing.
- **Sparse indexing**: Only every Nth key is indexed, trading O(block_size) scan cost for dramatically smaller index memory. This is the same tradeoff as in LevelDB's block index.
- **Heap-based k-way merge**: The standard algorithm for external merge sort, using Python's `heapq` with a tuple that encodes both sort key and tiebreaker.
- **Tombstone propagation**: Deletes are represented as entries with `value=None`, not as absence. This is essential — a delete must propagate through compaction to shadow older versions in lower levels. Only at the bottom (`remove_tombstones=True`) can they be dropped.
- **Strategy pattern**: `CompactionManager` dispatches to STCS or LCS methods via string-based strategy selection.

## Dependencies

**Imports:** All stdlib — `heapq` for k-way merge, `struct` for binary encoding, `os` for file size, `time` for path generation, `dataclasses` and `typing` for structure.

**Imported by:** `test_sstable.py` and `tester_test_sstable.py` — only test harnesses. This module is a leaf in the dependency graph with no downstream production consumers.

## Flow

**Write path:**
1. Caller creates `SSTableWriter`, calls `add()` for each key in sorted order.
2. Every `block_size` entries, the sparse index records the key and byte offset.
3. `finish()` appends the index block, footer, patches the header, and closes the file.

**Read path (point lookup):**
1. `SSTableReader.__init__` reads header, footer, and sparse index into memory.
2. `get(key)` binary-searches the index to find the block, then linearly scans within the block.
3. Returns the entry if found, `None` otherwise. If the key falls between index entries, scanning stops at the next index entry's offset.

**Compaction path:**
1. `CompactionManager.needs_compaction()` checks trigger conditions.
2. `run_compaction()` selects which SSTables to merge, calls `merge_sstables()`, updates the internal list, and returns the new SSTables.
3. For LCS, merged output gets `level = target_level` assigned.

## Invariants

- **Sorted key order within each SSTable** — `SSTableWriter.add()` does not enforce this; it's a caller responsibility. Violation would silently corrupt binary search in `get()`.
- **Tombstone encoding is unambiguous** — the `0xFF` marker works because it's the first byte of the value-length field; no valid 4-byte big-endian length can start with `0xFF` and represent a valid value length (it would mean a 4GB+ value). *However*, this is subtly fragile: if `value_len` could legitimately have `0xFF` as its high byte, the parser would misinterpret it.
- **Newest timestamp wins during merge** — the heap sorts by `(key, -timestamp)`, so for duplicate keys, the newest entry is popped first and all others are skipped.
- **L0 SSTables may have overlapping key ranges; L1+ must not** — LCS relies on this for correctness, but the code doesn't explicitly enforce non-overlap at L1+. The merge process produces a single output file, so within one compaction it's fine, but across multiple compactions the invariant relies on correct overlap detection.
- **Header is patched at close** — an SSTable whose `finish()` is never called has `entry_count=0` in the header, making it appear empty.

## Error Handling

Minimal. The reader asserts `magic == MAGIC` on open — the only validation. There is:
- No checksum or CRC on entries or blocks.
- No validation that keys are actually sorted.
- No file locking or crash recovery.
- No cleanup of old SSTable files after compaction (they're removed from the in-memory list but not deleted from disk).
- `_read_entry` returns `None` on truncated reads rather than raising, silently stopping iteration on corrupted files.

This is appropriate for a teaching implementation — it shows the algorithms clearly without the defensive engineering that production systems need.

## Topics to Explore

- [file] `sstable-and-compaction/test_sstable.py` — Understand the test scenarios covering write/read/merge/compaction, which reveal edge cases the implementation handles (or doesn't)
- [file] `log-structured-merge-tree/lsm.py` — The LSM tree implementation likely builds on SSTables with a memtable layer, showing how SSTables fit into a complete storage engine
- [function] `sstable-and-compaction/sstable.py:merge_sstables` — The k-way merge is the algorithmic core of compaction; trace through with duplicate keys and tombstones to understand resolution semantics
- [general] `tombstone-gc-safety` — When is it safe to remove tombstones? The `remove_tombstones` flag exists but the compaction manager never sets it to `True` — explore what conditions make tombstone removal correct
- [file] `hash-index-storage/bitcask.py` — Compare with the Bitcask-style log-structured hash index, which solves the same write-optimized storage problem with a different read strategy (hash index vs. sorted merge)

## Beliefs

- `sstable-tombstone-encoding` — Tombstones are encoded as a single `0xFF` byte; this works because valid value-length fields are 4 bytes big-endian, so no legitimate value starts with `0xFF` at the wire level
- `sstable-merge-lww` — K-way merge resolves duplicate keys by last-write-wins: the heap sorts by `(key, -timestamp)` so the newest entry is consumed first and all older duplicates are skipped
- `sstable-sparse-index-granularity` — The sparse index stores one entry per `block_size` keys (default 64), meaning point lookups scan up to 64 entries after the binary search narrows to a block
- `sstable-range-scan-no-index` — `range_scan()` does not use the sparse index for seek; it scans from the beginning of the data section, making it O(N) regardless of the range position
- `sstable-compaction-no-tombstone-gc` — Neither STCS nor LCS compaction passes `remove_tombstones=True` to `merge_sstables`, so tombstones accumulate indefinitely across compaction rounds

