# Topic: The original Bitcask paper from Riak describes the hint file format, merge process, and keydir rebuild strategy that this implementation models

**Date:** 2026-05-29
**Time:** 06:43

No problem — I have enough from the observations to write a thorough explanation. Here's the entry:

---

# Bitcask Paper Concepts: Hint Files, Merge, and Keydir Rebuild

The original Bitcask paper (from Riak at Basho) defines three core mechanisms that make an append-only hash index viable for production: **hint files** that accelerate startup, a **merge process** that reclaims disk space, and a **keydir rebuild strategy** that reconstructs the in-memory index after a crash or restart. This codebase has two independent implementations that model these ideas, each making slightly different trade-offs.

## The Keydir: An In-Memory Hash Index

In the Bitcask paper, the **keydir** is the single in-memory hash table mapping every live key to its position on disk: which file, what byte offset, how many bytes to read. Both implementations mirror this directly.

In `hash-index-storage/bitcask.py:35`, the keydir is a `dict[str, KeyEntry]` where each `KeyEntry` (lines 21–25) records `file_id`, `offset`, `size`, and `timestamp`. Every `put` writes an append-only record and updates the keydir in one step (line 171). Every `get` does a single hash lookup then a single disk seek (line 175).

The `log-structured-hash-table/bitcask.py` variant uses a simpler index structure — `dict[str, tuple[str, int]]` mapping key to `(filepath, offset)` (line 42). It doesn't store record size or timestamp in the index, which means reads must parse the header at the offset to discover record boundaries. This is a conscious simplification: less memory per key, but the get path does more header parsing (lines 162–175).

The paper's key insight is that **reads are always a single disk seek** because the keydir tells you exactly where to look. Neither implementation ever scans a data file to satisfy a read — the hash table eliminates that entirely.

## The Merge Process (Compaction)

Bitcask files are append-only, so updates and deletes leave stale records behind. The paper describes a **merge process** that reads immutable (non-active) data files, keeps only the latest version of each key, and writes the live records into new files.

### `hash-index-storage/bitcask.py:194` — `compact()`

This implementation follows the paper closely:

1. **Only compact immutable files** — the active file is excluded (line 196–198). This is critical: the active file may be receiving concurrent writes.

2. **Scan immutable files and identify live records** — it reads each old file, skips tombstones (line 221), and for each key checks whether the keydir still points into an immutable file. If the keydir points to the active file for that key, the immutable record is stale (line 228–233). This is the paper's "latest-wins" rule.

3. **Write merged output** — live records are re-written into new data files (lines 244–275), respecting the `max_file_size` rotation limit. The keydir is updated in-place to point at the new locations (line 276).

4. **Generate hint files for merged output** — after writing each merged data file, a hint file is written alongside it (lines 261, 280). This is the paper's rule: hint files are a *product of merge*, not of normal writes.

5. **Delete old files** — the original immutable files are removed after the merged files and hint files are safely written (lines 283–290).

### `log-structured-hash-table/bitcask.py` — `compact()`

This variant adds **CRC integrity checking** (`zlib.crc32` in the header format, line 8) that the hash-index version omits. During compaction it verifies record integrity, which means corrupted stale records are silently dropped rather than propagated into merged output. It also has **auto-compaction** — when the count of frozen segments exceeds `auto_compact_threshold`, compaction triggers automatically inside `put()` (lines 157–160).

## Hint Files: Accelerating Startup

The paper's key contribution to startup performance is the **hint file**. Without hint files, rebuilding the keydir requires reading every byte of every data file — parsing headers, extracting keys, skipping values. With hint files, you read a much smaller file that contains only the metadata needed to populate the keydir, with no value data at all.

### Hint file format in `hash-index-storage/bitcask.py`

The hint entry format is defined at line 13:

```python
HINT_FORMAT = "<IQId"  # file_id(uint32), offset(uint64), size(uint32), timestamp(double)
```

Each hint entry is: `[file_id | offset | size | timestamp | key_size | key_bytes]`. This is 24 bytes of fixed header (`HINT_HEADER_SIZE`) plus 4 bytes for key length plus the key itself (lines 158–165). Crucially, **no value data appears in the hint file** — that's the whole point. A hint file for a data file with megabytes of values might be only kilobytes.

### Hint file format in `log-structured-hash-table/bitcask.py`

The simpler variant at line 12:

```python
HINT_ENTRY_FMT = "!II"  # key_size, offset
```

This stores only `key_size` and `offset` — 8 bytes of header per entry (line 13). No file_id (implied by the hint file's own name), no timestamp, no record size. This is more compact but means the index after hint-file loading has less metadata available than the other implementation.

### When hint files are created

Both implementations follow the paper's rule: **hint files are produced by the merge process, not by normal writes**. In `hash-index-storage/bitcask.py`, `_write_hint_file` is called at lines 261 and 280 — both inside `compact()`. The active file never gets a hint file during normal operation.

The `log-structured-hash-table` variant exposes `create_hint_files()` as an explicit operation (called in `test_hint_files` at test line 110), giving the caller control over when hint files are generated.

## Keydir Rebuild Strategy

On startup, the keydir must be reconstructed. The paper describes a two-path strategy that both implementations follow:

### `hash-index-storage/bitcask.py:108` — `_rebuild_index()`

```python
def _rebuild_index(self, file_ids: list[int]):
    for fid in sorted(file_ids):
        hint_path = self._hint_path(fid)
        if os.path.exists(hint_path):
            self._load_hint_file(fid)
        else:
            self._scan_data_file(fid)
```

The logic is: **for each data file, in order from oldest to newest, prefer the hint file if one exists; otherwise fall back to scanning the full data file**. Processing in sorted order matters because later files contain newer records — when the same key appears in both file 3 and file 7, the keydir ends up with the file 7 entry, which is correct.

- **Fast path** — `_load_hint_file` (line 141) reads only the hint file metadata. No value bytes are touched. This is O(number of keys), not O(data size).
- **Slow path** — `_scan_data_file` (line 118) reads the entire data file, parsing every record header and key, skipping value bytes with a seek. Tombstones (records with `val_size == 0`) remove keys from the keydir (lines 136–137).

### `log-structured-hash-table/bitcask.py:65` — `_recover()`

Same dual-path strategy (lines 72–78): check for a `.hint` companion file; if present, use `_load_hint_file`; otherwise `_scan_segment`. The scan path here additionally validates CRC checksums and stops scanning on corruption (lines 99–101), which provides crash recovery — a partial write at the tail of a segment is simply ignored.

### Crash recovery

The test suites validate the rebuild strategy explicitly. `test_crash_recovery` in `hash-index-storage/test_bitcask.py:92` deliberately deletes all hint files and reopens the store, forcing the slow-path scan. It verifies that overwrites and deletes are correctly reflected. `test_hint_files` (line 61) verifies the fast path: compact, close, reopen, and confirm all data is accessible through hint-file-based rebuild.

## Where the Implementations Diverge from the Paper

Both implementations simplify the paper in a few ways worth noting:

- **No CRC in the hash-index variant** — the paper includes checksums in every record and hint entry. Only the log-structured variant (`HEADER_FMT = "!III"` with CRC) implements this.
- **Single-process assumption** — the paper describes the keydir as a shared data structure across Erlang processes. These Python implementations are single-threaded with no locking.
- **Tombstone representation** — the paper uses a special tombstone marker. The hash-index variant uses an empty value string (`val_size == 0`), while the log-structured variant uses a sentinel byte sequence (`b"__BITCASK_TOMBSTONE__"`). The sentinel approach is safer — it distinguishes "deleted" from "value is legitimately empty."

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — The full merge implementation: how it determines liveness, rotates output files, writes hint files, and atomically swaps old files for new ones
- [function] `log-structured-hash-table/bitcask.py:compact` — Compare the CRC-validating, auto-triggering compaction variant against the simpler hash-index version
- [file] `hash-index-storage/tester_test_bitcask.py` — Contains dedicated tests for hint-file-based startup vs. hint-less recovery, useful for understanding the two rebuild paths in isolation
- [general] `bitcask-paper-comparison` — Read the original Bitcask paper (Sheehy & Smith, 2010) to compare the Erlang-specific details (keydir sharing via ETS tables, lock files, merge triggers) against these Python simplifications
- [function] `log-structured-hash-table/bitcask.py:create_hint_files` — Explicit hint file generation as a separate operation from compaction — different from the paper's model where hints are always a merge byproduct

## Beliefs

- `hint-files-produced-by-merge-only` — In `hash-index-storage/bitcask.py`, hint files are exclusively written inside `compact()` (lines 261, 280); normal `put`/`delete` operations never create hint files
- `keydir-rebuild-prefers-hint-files` — Both implementations check for a `.hint` file before scanning a `.data` file during index rebuild (`_rebuild_index` line 112, `_recover` line 76), making startup time proportional to key count rather than data size when hints exist
- `rebuild-order-is-oldest-to-newest` — Files are processed in sorted ID order during rebuild so that newer records for the same key overwrite older ones in the keydir, ensuring the latest value wins without explicit timestamp comparison
- `tombstone-semantics-differ-between-implementations` — The hash-index variant uses `val_size == 0` as a tombstone (line 136), while the log-structured variant uses a sentinel byte sequence `b"__BITCASK_TOMBSTONE__"` (line 7), meaning the hash-index variant cannot store empty-string values
- `only-immutable-files-are-compacted` — Both implementations exclude the active/current file from merge, ensuring that in-flight writes are never disrupted by the compaction process

