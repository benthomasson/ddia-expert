# File: log-structured-hash-table/tester_test_bitcask.py

**Date:** 2026-05-29
**Time:** 06:42

I'll work from the file content provided.

---

# `log-structured-hash-table/tester_test_bitcask.py`

## Purpose

This is the **validation test suite** for the `BitcaskStore` implementation — a log-structured hash table modeled after [Bitcask](https://en.wikipedia.org/wiki/Bitcask) (the storage engine described in DDIA Chapter 3). The `tester_` prefix distinguishes it from the project's own unit tests (`test_bitcask.py`); this file acts as an **external conformance harness** that verifies the implementation meets a specification, rather than testing internals.

It validates eight behavioral properties: basic CRUD, segment rotation, compaction, CRC integrity, hint-file recovery, partial-write tolerance, auto-compaction, and large-dataset correctness. Together they cover the full lifecycle of a Bitcask store from first write through crash recovery.

## Key Components

### Fixtures

| Fixture | What it provides |
|---------|-----------------|
| `store_dir` | A fresh `tempfile.TemporaryDirectory` that's automatically cleaned up. Every test gets an isolated filesystem. |
| `store` | A `BitcaskStore` opened in `store_dir` with `max_segment_size=100` and `auto_compact_threshold=100`. Calls `s.close()` on teardown. |

`max_segment_size=100` is deliberately small — it forces segment rotation after just a few writes, which lets the tests exercise multi-segment behavior without writing megabytes of data.

### Test Functions

1. **`test_example_usage`** — End-to-end smoke test. Exercises put/get, update-overwrites, delete-returns-None, segment rotation, compaction (asserts stale entries removed ≥ 2), iteration via `__iter__`, and crash recovery (close + reopen). This is the "golden path" test; if this passes, the store is broadly functional.

2. **`test_segment_rotation`** — Writes 20 records of ~10 bytes each into a store with 100-byte segments. Asserts `num_segments >= 2`. Pure structural test — no data-correctness check.

3. **`test_compaction_disk_usage`** — Writes the same key 10 times (creating 9 stale entries) plus 10 unique keys, then compacts. Asserts `total_disk_usage` decreased and all live data survived. This is the compaction *value proposition* test.

4. **`test_crc_corruption`** — Writes one record, locates it via `store._index` (reaching into internals), flips a byte at `offset + HEADER_SIZE + 2` (inside the payload, past the header), then asserts `get` raises `CorruptionError`. Validates the CRC32 integrity check.

5. **`test_hint_files`** — Writes data, compacts, creates hint files, closes, then reopens. Asserts all data is readable. Hint files let the store rebuild its in-memory index without scanning every segment — this test confirms they're sufficient for recovery.

6. **`test_partial_write_recovery`** — Manually appends a truncated record (header + key but no value) to the active segment file, then reopens the store. Asserts the good record survives and the partial one is silently ignored. This simulates a crash mid-`write()`.

7. **`test_auto_compaction`** — Uses `auto_compact_threshold=3` and `max_segment_size=50`. After 50 writes, asserts `num_segments <= 5` (compaction kept segment count bounded) and all data is readable.

8. **`test_large_dataset`** — 10,000 keys, update every other one, compact, verify. Stress test for correctness at scale.

## Patterns

**Spec-conformance harness pattern.** The `tester_` prefix convention appears across the repo (every implementation directory has one). These files test the *public contract* — they import only public names (`BitcaskStore`, `CorruptionError`, `HEADER_SIZE`, `HEADER_FMT`) and treat the implementation as a black box, except `test_crc_corruption` which reaches into `_index` and `test_partial_write_recovery` which reaches into `_active_path`.

**Fixture layering.** `store_dir` provides the temp directory; `store` builds on it. Tests that need non-default constructor args (custom `max_segment_size`, etc.) skip the `store` fixture and construct their own instance from `store_dir`, always closing manually.

**Close-reopen cycle** for crash recovery. Tests 1, 5, and 6 all follow the pattern: write → close → reopen from same directory → assert state persisted. This validates the on-disk format is self-describing.

**Binary-level fault injection.** Tests 4 and 6 don't mock anything — they directly manipulate the segment file bytes using `struct.pack` and raw file I/O to simulate real corruption and crash scenarios. This is more realistic than mocking and tests the actual binary format.

## Dependencies

### Imports
- **`os`** — imported but unused in the visible code (likely a leftover or used in deleted tests)
- **`struct`** — packing the header in `test_partial_write_recovery`
- **`tempfile`** — `TemporaryDirectory` for test isolation
- **`zlib`** — `crc32` to construct a valid header in the partial-write test
- **`pytest`** — test framework and fixtures
- **`bitcask`** — the implementation under test: `BitcaskStore`, `CorruptionError`, `HEADER_SIZE`, `HEADER_FMT`

### Imported by
Nothing imports this file — it's a test module, run by pytest.

## Flow

Each test follows the same arc:

1. **Setup**: Get a temp directory (via fixture or inline), construct a `BitcaskStore` with specific tuning parameters.
2. **Exercise**: Write keys, update, delete, compact, create hint files — whatever the test targets.
3. **Assert**: Check return values (`get`, `len`, `num_segments`, `total_disk_usage`), check exceptions (`CorruptionError`), or check post-recovery state.
4. **Teardown**: `store.close()` (fixture teardown or explicit call).

The partial-write test has an extra phase between exercise and assert: it **closes the store**, **mutates the segment file at the binary level**, then **reopens** — testing the recovery path rather than the write path.

## Invariants

1. **Get-after-put returns the latest value.** Every test that calls `put` then `get` on the same key asserts this, including after updates.
2. **Deleted keys return `None`.** After `delete("age")`, `get("age") is None` and `contains("age")` is false.
3. **Compaction preserves all live data.** Tests 1, 3, and 8 all assert that non-stale keys survive compaction with correct values.
4. **Compaction reduces disk usage.** Test 3 asserts `usage_after < usage_before`.
5. **CRC corruption raises `CorruptionError`.** Not silently returned, not a different exception.
6. **Partial writes are silently discarded on recovery.** The store truncates back to the last complete record.
7. **Close-reopen preserves all committed state.** The on-disk format (segments + optional hint files) is the complete source of truth.
8. **Auto-compaction bounds segment count.** With `threshold=3`, 50 writes produce at most 5 segments.

## Error Handling

The test suite validates two error paths in the implementation:

- **`CorruptionError`** — `test_crc_corruption` asserts this is raised (via `pytest.raises`) when payload bytes are flipped. The CRC32 in the header won't match the corrupted payload. The test verifies the exception type, not the message.

- **Partial write tolerance** — `test_partial_write_recovery` verifies the *absence* of errors. A truncated record (key written, value missing) must be silently skipped during segment replay on startup. The store must not crash, raise, or index the incomplete record.

No tests check for `KeyError` or similar — the `BitcaskStore.get` API uses `None`-return semantics rather than exceptions for missing keys.

---

## Topics to Explore

- [file] `log-structured-hash-table/bitcask.py` — The implementation under test; understanding `HEADER_FMT`, the segment rotation logic, and compaction algorithm will clarify why tests use specific byte offsets and size thresholds
- [file] `log-structured-hash-table/test_bitcask.py` — The project's own unit tests, likely testing internals and edge cases that the tester harness intentionally skips
- [file] `hash-index-storage/bitcask.py` — A separate Bitcask implementation in the repo; comparing the two reveals how the same DDIA concept can be implemented with different trade-offs
- [general] `bitcask-binary-format` — The on-disk record format (`HEADER_FMT`, CRC, key-length, value-length, payload) is central to tests 4 and 6; understanding the exact byte layout explains the magic offsets
- [function] `log-structured-hash-table/bitcask.py:compact` — The compaction algorithm determines what "stale entries removed" means and how segment files are rewritten, which directly affects tests 1, 3, 7, and 8

## Beliefs

- `tester-tests-public-contract` — `tester_test_*.py` files test the public API contract of each implementation; they import only public symbols (plus format constants needed for fault injection)
- `bitcask-get-returns-none-for-missing` — `BitcaskStore.get` returns `None` for missing or deleted keys rather than raising an exception
- `bitcask-crc32-raises-corruption-error` — Reading a record whose payload doesn't match its CRC32 header raises `CorruptionError`, not a silent wrong value
- `bitcask-partial-write-silent-discard` — On recovery, records truncated mid-write (incomplete payload) are silently discarded without raising errors or appearing in the index
- `bitcask-compaction-preserves-live-data` — Compaction removes stale and tombstoned entries but every live key-value pair survives with its correct latest value

