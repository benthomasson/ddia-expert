# File: log-structured-hash-table/test_bitcask.py

**Date:** 2026-05-28
**Time:** 18:55

I'll work from the test file content you provided, which reveals enough about the implementation to explain thoroughly.

---

# `log-structured-hash-table/test_bitcask.py`

## 1. Purpose

This is the primary test suite for a **Bitcask-style log-structured hash table** — the storage engine described in Chapter 3 of *Designing Data-Intensive Applications*. It validates `BitcaskStore`, a key-value store that writes all mutations as append-only log entries and maintains an in-memory hash index mapping keys to on-disk offsets.

The file owns the correctness contract for the entire storage engine: CRUD operations, segment management, compaction, crash recovery, data integrity (CRC), and partial-write tolerance.

## 2. Key Components

### Fixtures

- **`store_dir`** — Creates a temporary directory per test, cleaned up automatically. Every test gets filesystem isolation.
- **`store`** — Instantiates a `BitcaskStore` with `max_segment_size=100` (deliberately small to trigger segment rotation in basic tests). Calls `close()` on teardown.

Several tests (`test_compaction`, `test_disk_usage_decreases`, `test_crash_recovery`, etc.) create their own `BitcaskStore` instances directly rather than using the fixture, because they need to control `auto_compact_threshold` or simulate crash-and-reopen scenarios.

### Test Cases (14 total)

| Test | What it validates |
|------|-------------------|
| `test_basic_put_get` | Put/get/miss semantics — `get` returns `None` for missing keys |
| `test_update` | Last-writer-wins: overwriting a key returns the newest value |
| `test_delete` | Tombstone semantics: `delete` returns `True` on first call, `False` on repeat |
| `test_segment_rotation` | Writing more data than `max_segment_size` creates multiple segment files |
| `test_compaction` | Compact removes stale entries (old overwrites, tombstones) while preserving live data |
| `test_disk_usage_decreases` | `total_disk_usage` drops after compaction — tests the actual on-disk effect, not just logical correctness |
| `test_crash_recovery` | Close + reopen rebuilds the in-memory index from segment files on disk |
| `test_hint_files` | After compaction, hint files enable fast index rebuilding without scanning full segments |
| `test_crc_corruption` | Corrupting a single byte in the payload triggers `CorruptionError` on read |
| `test_iteration` | `__iter__` yields live (key, value) pairs, skipping deleted keys |
| `test_large_dataset` | 10K keys with overwrites and compaction — a stress/integration test |
| `test_contains` | `contains()` method checks key existence without reading the value |
| `test_auto_compaction` | When segment count exceeds `auto_compact_threshold`, compaction fires automatically |
| `test_partial_write_recovery` | A truncated record at the end of a segment file is silently skipped on recovery |

## 3. Patterns

**Fixture-per-concern**: The `store` fixture handles the common case (small segments, auto-close). Tests that need crash simulation or custom thresholds manage their own lifecycle — this avoids a combinatorial fixture matrix.

**Progressive complexity**: Tests are numbered 1–14 and build from simple CRUD to crash recovery, corruption detection, and partial-write tolerance. This ordering mirrors how you'd validate a storage engine: basic correctness first, then durability, then adversarial edge cases.

**Direct filesystem manipulation**: Tests 9 and 14 bypass the `BitcaskStore` API and write raw bytes to segment files to simulate corruption and partial writes. This tests the defensive boundary between the store and the OS — exactly where real-world failures occur.

**Close-and-reopen pattern** (tests 7, 8, 14): The test writes data, closes the store, then opens a fresh `BitcaskStore` on the same directory. This simulates a process crash and validates that all durable state lives on disk, not just in memory.

## 4. Dependencies

**Imports:**
- `os`, `struct`, `tempfile`, `zlib` — stdlib for filesystem ops, binary packing, and CRC computation
- `pytest` — test framework and fixtures
- `bitcask.BitcaskStore` — the system under test
- `bitcask.CorruptionError` — custom exception for CRC mismatches
- `bitcask.HEADER_SIZE`, `bitcask.HEADER_FMT` — binary format constants shared between production and test code (the test uses these to craft valid-but-truncated records in `test_partial_write_recovery`)

**Imported by:** Nothing — this is a leaf test module. The sibling `tester_test_bitcask.py` likely runs a separate validation pass (a "tester-tests-the-test" pattern seen across this repo).

## 5. Flow

Each test follows the same arc:

1. **Setup**: Get a temp directory (fixture or inline), create a `BitcaskStore`
2. **Write**: Put keys/values through the store's API
3. **Optionally mutate**: Overwrite, delete, corrupt, or truncate
4. **Optionally cycle**: Close and reopen to test durability
5. **Assert**: Verify reads return expected values (or `None`/`CorruptionError`)
6. **Teardown**: Close the store, temp directory is cleaned up

The CRC corruption test (`test_crc_corruption`) has a notable flow:
- Writes a record, then reaches into the store's `_index` (a private dict mapping keys to `(segment_path, offset)` tuples) to find the exact byte offset
- Seeks past `HEADER_SIZE` + 2 bytes into the payload area and overwrites one byte with `0xFF`
- On the next `get()`, the store recomputes the CRC over the read payload, finds it doesn't match the stored CRC in the header, and raises `CorruptionError`

The partial-write test (`test_partial_write_recovery`) manually constructs a record header using `struct.pack(HEADER_FMT, crc, key_len, value_len)`, writes the key bytes but *not* the value bytes, then verifies the reopened store ignores this truncated tail.

## 6. Invariants

- **Last-writer-wins**: The most recent `put` for a key is the value returned by `get` — enforced by `test_update` and `test_compaction`.
- **Delete is a tombstone**: After `delete`, the key reads as `None`. Compaction removes the tombstone. Repeated delete returns `False`.
- **Compaction is lossless for live data**: Every live key/value survives compaction unchanged (`test_compaction`, `test_large_dataset`).
- **Compaction reduces disk usage**: `total_disk_usage` strictly decreases when stale entries exist (`test_disk_usage_decreases`).
- **CRC guards every record**: A single-byte flip in the payload region raises `CorruptionError` — no silent data corruption.
- **Partial writes are safe**: A truncated record at segment tail is skipped on recovery; previously committed records remain readable.
- **Segment rotation triggers at `max_segment_size`**: Writing beyond the threshold creates a new segment file.
- **Auto-compaction bounds segment count**: When `num_segments` exceeds `auto_compact_threshold`, compaction fires automatically.

## 7. Error Handling

- **`CorruptionError`**: Raised by `get()` when the CRC computed from the read payload doesn't match the CRC stored in the record header. This is a hard error — the test expects it to propagate as an exception, not be silently swallowed.
- **Partial writes**: Handled silently during recovery (startup). The store detects that the remaining bytes in the file are fewer than the payload size declared in the header, and skips the incomplete record. No exception is raised — this is expected after a crash.
- **Missing keys**: Return `None` from `get()`, not an exception. This is the standard Bitcask contract.
- **Double delete**: Returns `False` (idempotent) rather than raising.

---

## Topics to Explore

- [file] `log-structured-hash-table/bitcask.py` — The implementation: how the append-only log, in-memory hash index, CRC framing, and compaction actually work
- [function] `log-structured-hash-table/bitcask.py:compact` — The compaction algorithm that rewrites live entries into new segments and updates the index atomically
- [file] `log-structured-hash-table/tester_test_bitcask.py` — The meta-test layer: how this repo validates that the test suite itself is correct and comprehensive
- [file] `hash-index-storage/bitcask.py` — The sibling `hash-index-storage` implementation — compare the two Bitcask variants to understand what design tradeoffs each makes
- [general] `bitcask-paper` — The original Bitcask paper from Riak describes the hint file format, merge process, and keydir rebuild strategy that this implementation models

## Beliefs

- `bitcask-tombstone-semantics` — `BitcaskStore.delete()` writes a tombstone record; `get()` returns `None` for tombstoned keys, and `compact()` removes both the original entry and the tombstone
- `bitcask-crc-per-record` — Every record has a CRC32 in its header (`HEADER_FMT` includes the checksum field), and `get()` raises `CorruptionError` on mismatch rather than returning corrupt data
- `bitcask-partial-write-safe` — On startup, the store skips incomplete records at segment tail (header present but payload truncated) without raising errors or losing previously committed data
- `bitcask-auto-compact-threshold` — When the number of segments exceeds `auto_compact_threshold`, compaction is triggered automatically during `put()` operations, reducing segment count
- `bitcask-index-is-memory-only` — The in-memory `_index` dict maps keys to `(segment_path, offset)` tuples and is rebuilt from disk on startup; hint files accelerate this rebuild by avoiding full segment scans

