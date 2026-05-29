# Topic: How real Bitcask implementations (Riak's original, or modern forks) mitigate the O(n) startup scan through hint files, memory-mapped I/O, and parallel file scanning

**Date:** 2026-05-29
**Time:** 06:52

Let me work with what I've gathered from the knowledge base entries. I have comprehensive coverage of the two implementations and their startup strategies.

---

# How Bitcask Mitigates the O(n) Startup Scan

## The Problem

Bitcask stores everything in an append-only log, but its read path depends on an **in-memory hash index** (the "keydir") that maps every key to its exact location on disk. This index is never persisted as a standalone data structure during normal writes — so every time the store opens, it must be reconstructed. Naively, that means reading every byte of every data file: parsing headers, extracting keys, skipping value payloads. For a store with gigabytes of values, this is an O(data_size) scan that blocks the store from serving reads until it completes.

The codebase contains **two independent Bitcask implementations** that demonstrate the primary mitigation strategy from the original Bitcask paper (Sheehy & Smith, 2010), and also reveal what they *don't* implement.

## Strategy 1: Hint Files (Implemented)

Hint files are the core optimization. They are compact companion files (`.hint` alongside `.data`) that contain only the metadata needed to populate the keydir — **no value data at all**. This reduces startup from O(data_size) to O(key_count × key_size).

### How the dual-path rebuild works

Both implementations follow the same pattern during startup: check for a hint file first, fall back to a full data scan if none exists.

**`hash-index-storage/bitcask.py` — `_rebuild_index()`** (documented in `entries/2026/05/28/hash-index-storage-bitcask-_rebuild_index.md`):

```python
def _rebuild_index(self, file_ids: list[int]):
    for fid in sorted(file_ids):
        if os.path.exists(self._hint_path(fid)):
            self._load_hint_file(fid)      # Fast path: O(keys)
        else:
            self._scan_data_file(fid)      # Slow path: O(data_size)
```

**`log-structured-hash-table/bitcask.py` — `_recover()`** (documented in `entries/2026/05/29/log-structured-hash-table-bitcask-_recover.md`):

```python
def _recover(self):
    segments = _find_existing_segments()  # Sorted by ID
    for seg_path in segments:
        if os.path.exists(_hint_path(seg_path)):
            self._load_hint_file(hint_path, seg_path)  # Fast path
        else:
            self._scan_segment(seg_path)                # Slow path
```

Files are processed **oldest-to-newest** (ascending ID order). This is critical: when the same key appears in multiple files, the latest file's entry overwrites earlier ones in the keydir, giving last-writer-wins semantics without explicit timestamp comparison.

### What hint files contain (and what they skip)

The hint file formats differ between the two implementations, but the principle is identical: store only `(key, file_location)` metadata, never store values.

**Hash-index variant** — 24-byte fixed header per entry:

```
HINT_FORMAT = "<IQId"   # file_id(u32) + offset(u64) + size(u32) + timestamp(f64)
```

Each entry: `[header (24 bytes)] [key_len (4 bytes)] [key_bytes (variable)]`.
Total per entry: 28 bytes + key length. A data file with megabytes of values produces a hint file of kilobytes.

**Log-structured variant** — 8-byte fixed header per entry:

```
HINT_ENTRY_FMT = "!II"  # key_size(u32) + offset(u32)
```

Each entry: `[header (8 bytes)] [key_bytes (variable)]`. Even more compact — no file_id (implied by which hint file you're reading), no timestamp, no record size.

### How hint files are produced

Hint files are **only created during compaction**, never during normal writes. This is a direct implementation of the paper's model.

In `hash-index-storage/bitcask.py`, `compact()` (line 194) calls `_write_hint_file()` at two points: when a merged data file reaches `max_file_size` and triggers rotation, and after the final merged file is closed. The entries are accumulated during the merge-write loop and reflect the new file's offsets.

The `log-structured-hash-table` variant exposes `create_hint_files()` as a separate explicit operation that can be called independently of compaction — a design divergence from the paper.

### Why hint files have no tombstones

A subtle but important detail: **hint files never contain tombstone records**. Compaction already filters out deleted keys before writing, so the hint file writer never sees them. This means `_load_hint_file` can unconditionally insert entries into the keydir without checking for deletions — documented as the `tombstone-handling-asymmetry` between the fast and slow paths.

### The slow-path fallback

When no hint file exists (the file was never compacted, or the hint file was lost/corrupted), the store falls back to a full data-file scan.

**`_scan_data_file`** in the hash-index variant reads every record's 16-byte header (`timestamp:f64, key_size:u32, value_size:u32`), reads the key bytes, **skips the value bytes** (seeks past them), and builds keydir entries. Tombstones (`val_size == 0`) trigger removal of the key from the index. This is O(data_size) because even though values are skipped, the scan must parse every header to know where the next record starts.

**`_scan_segment`** in the log-structured variant adds **CRC32 validation** — it computes a checksum over each record's payload and compares it to the stored CRC. On mismatch (corruption or partial write from a crash), it stops scanning entirely. This is the crash-recovery mechanism: partial writes at the tail of a segment are silently discarded.

## Strategy 2: Memory-Mapped I/O (Not Implemented)

Neither implementation uses `mmap`. Both use standard `open()` / `read()` file operations with manual buffering. The `_load_hint_file` in the hash-index variant reads the entire hint file into memory as a single `bytes` object and walks it with a position cursor — this is effectively a userspace equivalent of `mmap` for small files, but without the kernel-level page cache optimizations that `mmap` would provide for larger files.

For a reference implementation this is fine. In production Bitcask implementations (Riak's Erlang version, or modern forks like `bitcask-go`), `mmap` accelerates both hint file loading and data file scanning by:

- Eliminating `read()` syscall overhead — pages are faulted in on access
- Leveraging the OS page cache without explicit buffering
- Allowing random access without seek overhead (relevant when reading records out of order during compaction)

The absence of `mmap` here is intentional simplicity, not an oversight.

## Strategy 3: Parallel File Scanning (Not Implemented)

Both implementations process files **strictly sequentially** in the rebuild loop. There is no threading, multiprocessing, or async I/O during startup.

```python
for fid in sorted(file_ids):    # One at a time, in order
    ...
```

This is constrained by the **single-writer assumption** that runs through both implementations: no file locking, no synchronization primitives, no concurrent read safety. The knowledge base confirms this explicitly via the `compact-single-threaded-assumption` and `put-not-thread-safe` beliefs.

In production Bitcask, parallel scanning is viable because:

1. **The hint file fast path is embarrassingly parallel.** Each hint file is self-contained — it describes one data file's keys. Multiple hint files can be loaded concurrently, with results merged into the keydir using last-writer-wins on file ID ordering. The merge step requires a barrier (all files must finish before the keydir is consistent), but the I/O is parallelizable.

2. **The slow path is less obviously parallel.** Data file scans modify the keydir as they go, and tombstones in file N can delete entries from file N-1. If you parallelize, you need a merge step that respects ordering — scan files independently, then apply results to the keydir in ID order. This is doable but more complex.

3. **Riak's implementation** (Erlang) used concurrent hint file loading across schedulers and merged results into the ETS (Erlang Term Storage) keydir, which supports concurrent writes from multiple processes.

## What the Implementations Get Right

Despite the missing optimizations, the implementations nail the **architectural invariants** that make hint files correct:

- **Hint files are a product of compaction only** — they are never stale relative to the compacted output because they're written atomically with it.
- **Ordering determines correctness** — ascending file ID processing means no timestamps or version numbers are needed for conflict resolution during rebuild.
- **The hint path is optional** — every rebuild can fall back to the slow path, so a lost or corrupt hint file costs performance, not correctness.
- **Hint files skip values entirely** — the O(keys) vs O(data_size) distinction is preserved.

## What's Missing for Production Use

| Gap | Impact | Production Fix |
|-----|--------|---------------|
| No `mmap` | Higher syscall overhead, no page cache sharing | Memory-map data and hint files |
| No parallelism | Startup time scales linearly with file count | Parallel hint file loading + ordered merge |
| No hint CRC | Corrupt hint file → wrong index, no fallback | Add checksums to hint format; fall back to scan on mismatch |
| No compaction manifest | Crash during compaction can lose data | Write a `MANIFEST` file listing active segments atomically |
| No `fsync` on hint files | Hint file lost on crash → slow restart | `fsync` hint files, or accept the cost |

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:_load_hint_file` — The fast-path loader that reads the entire hint file into memory and parses it with a manual cursor; compare its I/O pattern to what `mmap` would provide
- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — The CRC-validating slow-path scanner; the only implementation with integrity checking during rebuild, which is what production Bitcask implementations apply to hint files too
- [general] `bitcask-parallel-hint-loading` — How Riak's Erlang Bitcask used ETS tables and concurrent schedulers to parallelize hint file loading across cores during startup
- [file] `log-structured-hash-table/bitcask.py` — The simpler hint format (`!II` — 8 bytes per entry vs 28+) illustrates the minimum metadata needed to reconstruct a keydir, at the cost of less information available post-recovery
- [general] `leveldb-manifest-pattern` — LevelDB's `MANIFEST` / `CURRENT` file pattern solves the compaction atomicity gap; understanding it clarifies what these implementations trade away for simplicity

---

## Beliefs

- `hint-file-converts-startup-from-data-proportional-to-key-proportional` — With hint files, startup time is O(number_of_keys × avg_key_size) instead of O(total_data_size), because hint files contain no value payloads; the hash-index variant stores 28 + key_length bytes per entry vs. the full record with arbitrarily large values
- `hint-files-are-compaction-only-and-optional` — Hint files are produced exclusively during compaction and are never required for correctness; a missing or corrupt hint file triggers a transparent fallback to full data-file scanning with no data loss
- `no-parallel-startup-due-to-single-writer-assumption` — Both implementations process files sequentially during index rebuild because they share the single-writer, no-synchronization architecture; production Bitcask implementations parallelize hint loading across cores with a final ordered merge
- `hint-no-integrity-validation` — Neither implementation validates hint file integrity (no CRC, no magic bytes, no version field); a corrupt hint file produces wrong keydir entries that silently serve incorrect data, whereas the log-structured variant's `_scan_segment` at least has CRC validation on the slow path
- `rebuild-ordering-is-the-sole-correctness-mechanism` — Both `_rebuild_index` and `_recover` rely entirely on processing files in ascending ID order to resolve key conflicts; there is no explicit version counter, vector clock, or conflict resolution — the last file scanned wins

