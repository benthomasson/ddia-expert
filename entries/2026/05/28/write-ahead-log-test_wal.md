# File: write-ahead-log/test_wal.py

**Date:** 2026-05-28
**Time:** 18:15

# `write-ahead-log/test_wal.py`

## Purpose

This is the test suite for a Write-Ahead Log (WAL) implementation — the durability primitive described in DDIA Chapter 3. The file validates that `WriteAheadLog` correctly handles the core WAL contract: every mutation is durably recorded before it's acknowledged, and the log can be replayed to recover state after a crash. It owns the behavioral specification of the WAL through 9 test functions covering the happy path, failure modes, and edge cases.

## Key Components

Each test function exercises a distinct WAL capability:

| Function | What it validates |
|---|---|
| `test_basic` | Full lifecycle: single appends, batch append, checkpoint, replay filtering by sequence number |
| `test_crash_recovery` | Simulates a crash by opening a second `WriteAheadLog` instance on the same directory without closing the first — verifies records survive |
| `test_corruption` | Overwrites the last 5 bytes of a WAL file with `0xFF`, then confirms replay recovers only the uncorrupted prefix (1 of 2 records) |
| `test_truncation` | Calls `wal.truncate(2)` to discard records with `seq <= 2`, then verifies only seq=3 survives |
| `test_rotation` | Sets `max_file_size=100` to force multiple `.wal` files, then checks all 20 records are readable across segments |
| `test_iterate` | Verifies the iterator interface and that `append_batch` produces a trailing `COMMIT` record (4 records total for 1 single + 1 batch of 2) |
| `test_empty` | Edge case: fresh WAL returns no records and seq=0 |
| `test_sync_modes` | Smoke test — constructs a WAL in each of `"sync"`, `"batch"`, `"none"` modes without error |
| `test_large_values` | Writes a 100KB value and confirms round-trip fidelity |

## Patterns

**No test framework.** Tests are plain functions with `assert` statements, run via `__main__`. Each prints `PASSED: ...` on success. This is consistent across the DDIA implementations repo — the tests are self-contained scripts, not pytest suites.

**Temp directory isolation.** Every test uses `tempfile.TemporaryDirectory()` as a context manager, so WAL files are cleaned up automatically and tests don't interfere with each other.

**Crash simulation via re-instantiation.** `test_crash_recovery` doesn't kill a process — it opens a fresh `WriteAheadLog` on the same directory while the first is still open. This tests that the WAL's on-disk format is self-describing enough to recover without a clean shutdown.

**Physical corruption injection.** `test_corruption` seeks to 5 bytes before EOF and writes garbage. This tests the WAL's checksum/integrity mechanism — the implementation must detect the corrupted record and stop replay before it.

## Dependencies

**Imports:**
- `wal.WriteAheadLog` — the implementation under test (sibling file `wal.py`)
- `tempfile`, `os`, `glob` — filesystem utilities for test isolation and corruption injection
- `sys` — imported but unused

**Imported by:**
- `tester_test_wal.py` — likely a wrapper that runs these tests in a harness or validates their output

## Flow

When run as `python test_wal.py`, execution is sequential through all 9 tests. Each test:

1. Creates an isolated temp directory
2. Instantiates `WriteAheadLog(dir, sync_mode="sync")` — the strictest durability mode
3. Performs writes (`append`, `append_batch`)
4. Performs reads (`replay`, `iterate`, `current_seq_num`)
5. Asserts expected record counts, sequence numbers, and field values
6. Calls `wal.close()` and lets the context manager clean up

The sequence number contract visible from the tests: `append` returns the assigned seq, starting at 1. `append_batch` of 3 items advances the seq by 4 (3 data records + 1 commit marker), bringing the total to 7 in `test_basic`. `checkpoint` writes a marker at seq=8.

## Invariants

- **Monotonic sequence numbers**: seq starts at 1 and increments by 1 per record (including batch items and commit markers).
- **Checkpoint filters replay**: `wal.replay(after_seq=cp_seq)` returns only records written after the checkpoint.
- **Corruption stops replay at the boundary**: corrupting the last record yields exactly 1 recovered record, not 0 and not 2. The WAL must not skip corrupted records and continue.
- **Truncation is inclusive**: `truncate(2)` removes records with seq <= 2, keeping seq >= 3.
- **Batch produces a COMMIT record**: `append_batch` of N items produces N+1 records (N data + 1 COMMIT), and `COMMIT` has `op_type == "COMMIT"`.
- **Rotation is transparent**: replay across multiple segment files returns the same records as if written to a single file.

## Error Handling

The tests don't exercise explicit error paths (no exception assertions). Error handling is tested implicitly:
- **Corruption** — the WAL is expected to silently truncate at the corruption point rather than raise. The test asserts `len(records) == 1`, meaning the implementation stops at the first bad checksum.
- **No `try/except`** — a failing assert crashes the script with a traceback. The `f"expected X, got {len(records)}"` messages on assertions aid debugging when they fire.

## Topics to Explore

- [file] `write-ahead-log/wal.py` — The implementation: how records are serialized, checksummed, and fsynced; how replay detects corruption
- [function] `write-ahead-log/wal.py:append_batch` — Understand the atomic batch + COMMIT marker protocol and how it interacts with crash recovery
- [function] `write-ahead-log/wal.py:truncate` — How truncation is implemented on a segmented log (delete whole files? rewrite?)
- [file] `write-ahead-log/tester_test_wal.py` — The test harness wrapper; likely validates output format or runs tests in isolation
- [general] `wal-checksum-format` — What integrity scheme the WAL uses (CRC32, SHA, length prefix) — critical for understanding `test_corruption`'s 5-byte overwrite strategy

## Beliefs

- `wal-batch-adds-commit-record` — `append_batch` of N items writes N+1 records: the N data records plus a trailing record with `op_type == "COMMIT"`
- `wal-corruption-truncates-at-boundary` — When a WAL file has a corrupted record, `replay()` returns all valid records before the corruption and silently discards the rest
- `wal-seq-starts-at-one` — Sequence numbers begin at 1 for the first appended record; an empty WAL reports `current_seq_num() == 0`
- `wal-truncate-is-inclusive` — `truncate(seq)` removes all records with sequence number less than or equal to `seq`
- `wal-checkpoint-seq-is-next-after-data` — A checkpoint after 7 data records occupies seq=8, and `replay(after_seq=8)` returns nothing written before the checkpoint

