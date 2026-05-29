# File: log-structured-merge-tree/lsm.py

**Date:** 2026-05-28
**Time:** 18:09

# `log-structured-merge-tree/lsm.py`

## Purpose

This file implements a **Log-Structured Merge Tree (LSM Tree)** storage engine — the write-optimized data structure described in Chapter 3 of *Designing Data-Intensive Applications*. It owns the full read/write path: ingesting key-value pairs through an in-memory buffer (memtable), persisting them to sorted on-disk files (SSTables), and merging those files via compaction. This is the same fundamental architecture behind LevelDB, RocksDB, and Cassandra's storage layer.

## Key Components

### `TOMBSTONE = b""`

A sentinel empty-bytes value representing a deleted key. Deletions are writes — the tombstone propagates through memtable flushes and is only garbage-collected during compaction.

### `WAL` (Write-Ahead Log)

Durability layer. Every mutation hits the WAL before the memtable, so an in-memory crash loses nothing.

- **`append(key, value)`** — writes a length-prefixed key and value to the log file, then `flush()`es. The wire format is `[4-byte key length][key bytes][4-byte value length][value bytes]`, big-endian.
- **`replay()`** — reads the entire WAL file and returns `(key, value)` pairs. Tolerates truncated entries at the tail (partial writes from a crash) by breaking out of the parse loop when any length/data read falls short.
- **`truncate()`** — zeros the WAL by reopening in `"wb"` mode, then reopens for appending. Called after a successful memtable flush.

### `SSTable` (Sorted String Table)

Immutable on-disk file containing key-value pairs in sorted order, with a **sparse index** embedded in a footer for efficient lookups.

**File layout:**

```
[data entries: (klen, key, vlen, value)*]
[footer: (klen, key, offset)* for every Nth entry]
[footer_start: 8 bytes]
[index_count: 4 bytes]
```

- **`SSTable.write(path, seq, entries, ...)`** — static factory. Writes sorted entries to disk, building the sparse index every `sparse_index_interval` entries (default 16). Returns a ready-to-query `SSTable`.
- **`load_index()`** — reads the footer from an existing file. Seeks to the last 12 bytes to find `footer_start` and `count`, then reads the sparse index entries.
- **`get(key)`** — binary searches the sparse index via `bisect.bisect_right` to find the candidate block, then does a linear scan within that block. Returns `(found: bool, value: Optional[bytes])`.
- **`scan(start_key, end_key)`** / **`scan_all()`** — sequential iteration for range queries and compaction.

### `LSMTree`

The top-level storage engine coordinating all components.

- **`put(key, value)`** — WAL append → memtable insert → conditional flush if memtable reaches `memtable_threshold`.
- **`get(key)`** — searches memtable → immutable memtables → SSTables (all newest-first). Returns the first match, or `None` if the key is absent or tombstoned.
- **`delete(key)`** — writes a tombstone via the same put path.
- **`range_scan(start_key, end_key)`** — merges results from all levels using a `dict` keyed by `(key → (priority, value))`, where higher priority means newer. Filters tombstones from the final result.
- **`_flush()`** — freezes the current memtable, writes it as an SSTable, truncates the WAL, and triggers compaction if the SSTable count reaches `compaction_threshold`.
- **`compact()`** — full compaction: collects all entries from all SSTables, sorts by `(key, -seq)` to ensure newest-wins deduplication, drops tombstones, writes a single new SSTable, and deletes the old files.

## Patterns

**Write-optimized with read amplification tradeoff.** Writes are always sequential (append to WAL, insert into sorted memtable). Reads may touch memtable + N SSTables in the worst case — classic LSM tradeoff.

**Tombstone-based deletion.** Deletes never remove data in place. The tombstone must survive through all levels until compaction merges it out. This is why `get()` checks `v == TOMBSTONE` and returns `None` rather than the bytes.

**Sparse indexing.** Rather than indexing every key (which would be large), the SSTable samples every 16th key. Lookups binary-search the sparse index then linear-scan a small block. This is the same approach described in DDIA's SSTable section.

**Sequence numbers for recency.** Each SSTable gets a monotonically increasing `seq`. During both point reads (`reversed(self._sstables)`) and compaction (`sort by -seq`), newer data always wins.

**Size-tiered compaction (simplified).** All SSTables are in a single "level" and compaction merges everything into one file. This is closer to size-tiered than leveled compaction — simpler but with higher space amplification.

## Dependencies

**Imports:**
- `struct` — binary encoding/decoding of the on-disk format (big-endian `>I` for 4-byte unsigned, `>Q` for 8-byte unsigned)
- `bisect` — binary search over the sparse index in `SSTable.get()`
- `sortedcontainers.SortedDict` — the memtable's backing structure, giving O(log n) insert and sorted iteration without manual balancing (stands in for a red-black tree)
- `heapq` — imported but **unused**; likely a leftover from an earlier k-way merge implementation in `compact()`
- `os` — filesystem operations (makedirs, listdir, remove, path joins)

**Imported by:**
- `test_lsm.py` — the test suite

## Flow

### Write Path
```
put(key, value)
  → WAL.append(key, encoded_value)
  → memtable[key] = encoded_value
  → if len(memtable) >= threshold:
      _flush()
        → freeze memtable, replace with empty SortedDict
        → SSTable.write(frozen entries)
        → WAL.truncate()
        → if len(sstables) >= compaction_threshold:
            compact()
```

### Read Path
```
get(key)
  → check memtable           (newest)
  → check immutable memtables (newest first)
  → check SSTables            (newest first, via sparse index + scan)
  → return first match, or None
```

### Recovery Path
```
LSMTree.__init__()
  → _load_existing_sstables()   (scan dir for .sst files, load their indexes)
  → WAL(path)
  → _replay_wal()               (re-populate memtable from WAL entries)
```

## Invariants

1. **WAL-before-memtable.** Every mutation writes to the WAL before the memtable, ensuring crash recovery can reconstruct the memtable state.
2. **SSTables are sorted by key internally** and tracked newest-last in `self._sstables` (sorted by `seq`).
3. **Newest data wins.** In `get()`, memtable is checked first; SSTables are traversed in reverse-seq order. In `range_scan()`, higher-priority sources overwrite lower. In `compact()`, entries are sorted by `(key, -seq)` and only the first occurrence (newest) is kept.
4. **Tombstones are only removed during compaction.** A tombstone in the memtable or an SSTable will shadow older values of the same key until `compact()` runs.
5. **WAL is truncated only after a successful SSTable write.** If the process crashes between WAL.append and flush, replay recovers the data. If it crashes after SSTable.write but before WAL.truncate, the WAL entries are harmlessly replayed into the memtable (idempotent because it's a dict).
6. **Sparse index interval is fixed per SSTable** at construction time.

## Error Handling

Error handling is minimal, consistent with a teaching implementation:

- **WAL replay** is crash-tolerant: any truncated record at the tail is silently skipped (`if pos + N > len(data): break`). This handles the case where a crash interrupted a write mid-entry.
- **SSTable reads** (`_read_entry`) return `None` on short reads, causing the caller's loop to break gracefully.
- **No checksums.** Corruption within a complete record (bit flips) would go undetected — a real LSM (like LevelDB) uses CRC32 per block.
- **No locking.** Concurrent access is not safe; this is a single-threaded implementation.
- **`_load_existing_sstables` assumes filename format** `sst_NNNNNN.sst` — a malformed filename would crash on `int(fname.split("_")[1].split(".")[0])`.

## Topics to Explore

- [file] `log-structured-merge-tree/test_lsm.py` — How the LSM tree is exercised: edge cases around tombstones, crash recovery, and compaction behavior
- [file] `sstable-and-compaction/sstable.py` — Compare with the standalone SSTable+compaction implementation to see if they share design or diverge (e.g., leveled vs size-tiered compaction)
- [file] `write-ahead-log/wal.py` — The dedicated WAL implementation; likely more complete than the WAL embedded here
- [function] `log-structured-merge-tree/lsm.py:range_scan` — The merge strategy uses a dict with priority overwrite rather than a proper k-way merge with a heap; worth understanding why `heapq` is imported but unused
- [general] `bloom-filter-integration` — Real LSM trees use bloom filters to skip SSTables that definitely don't contain a key; the `bloom-filter/` module in this repo is a natural extension

## Beliefs

- `lsm-wal-before-memtable` — Every `put()` and `delete()` writes to the WAL before inserting into the memtable, ensuring no acknowledged write is lost on crash
- `lsm-tombstone-is-empty-bytes` — Deletion is represented as `TOMBSTONE = b""` (empty bytes), which means empty-string values and deleted keys are indistinguishable at the storage layer
- `lsm-compaction-is-full-merge` — `compact()` merges all SSTables into a single new SSTable (size-tiered, single-level), not incremental or leveled
- `lsm-heapq-imported-unused` — `heapq` is imported but never referenced in the code; `compact()` uses list sorting instead of a heap-based k-way merge
- `lsm-sparse-index-default-16` — The SSTable sparse index samples every 16th key by default, meaning a point lookup may scan up to 16 entries after the binary search narrows to a block

