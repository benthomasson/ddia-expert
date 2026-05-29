# File: write-ahead-log/tester_test_wal.py

**Date:** 2026-05-28
**Time:** 18:47

# `write-ahead-log/tester_test_wal.py`

## Purpose

This is a **standalone integration test suite** for the Write-Ahead Log (WAL) implementation. The `tester_` prefix distinguishes it from `test_wal.py` — this file is designed to run directly via `python tester_test_wal.py` (note the `__main__` block), not through pytest. It validates the WAL's durability guarantees: that writes survive crashes, corruption is detected, and sequence numbers remain monotonic across restarts. It's the behavioral contract for `wal.py`, encoding the invariants a correct WAL must satisfy.

## Key Components

Each test function exercises one WAL guarantee:

| Function | Guarantee tested |
|----------|-----------------|
| `test_basic_append_and_replay` | Single writes, batch writes, checkpoint, and replay all round-trip correctly. Sequence numbers are assigned contiguously starting at 1. |
| `test_crash_recovery` | Opening a new `WriteAheadLog` on the same directory **without closing** the first recovers all fsynced records. Sequence numbering resumes correctly. |
| `test_corruption_stops_replay` | Binary corruption in the WAL file causes replay to **stop at the corruption boundary** — earlier valid records are still returned. |
| `test_truncation` | `truncate(n)` removes all records with `seq_num <= n`. Only records after the truncation point survive. |
| `test_log_rotation` | When `max_file_size` is exceeded, the WAL creates multiple `.wal` files. Replay spans all segments transparently. |
| `test_iterate_includes_commit` | `iterate()` yields raw records **including internal COMMIT markers**, unlike `replay()` which filters them. |
| `test_empty_wal` | A fresh WAL has sequence number 0 and replays nothing. |
| `test_large_values` | 100KB values survive serialization and deserialization intact. |
| `test_checkpoint_replay_after` | `replay(after_seq=cp)` returns only records written after the checkpoint sequence number. |
| `test_sequence_monotonic_across_restart` | After a clean close and reopen, the next sequence number continues from where the previous instance left off. |

## Patterns

**Temp-directory isolation.** Every test uses `tempfile.TemporaryDirectory()` as a context manager, so WAL files are cleaned up automatically and tests can't interfere with each other.

**Crash simulation by abandonment.** `test_crash_recovery` simulates a crash by simply not calling `wal.close()` and opening a second `WriteAheadLog` on the same directory. This works because `sync_mode="sync"` forces fsync on each write — the data is durable on disk even without a graceful shutdown.

**Direct binary corruption.** `test_corruption_stops_replay` opens the `.wal` file in binary mode and overwrites the last 5 bytes with `0xFF`. This is a targeted corruption that damages the final record while leaving the first record intact, testing the WAL's CRC/checksum validation.

**Print-on-pass reporting.** Each test prints its own pass message. The `__main__` block runs all tests sequentially and prints `ALL TESTS PASSED` at the end. No test framework dependencies — this is a self-contained harness.

## Dependencies

**Imports:**
- `wal.WriteAheadLog` — the system under test, expected to be in the same directory
- `tempfile`, `os`, `glob` — filesystem operations for test setup and corruption injection
- `sys` — imported but unused (likely leftover)

**Imported by:** Nothing directly. Run as a script or potentially invoked by a CI harness that discovers `tester_*.py` files.

## Flow

When run as `__main__`:

1. Each test creates an isolated temp directory
2. Instantiates `WriteAheadLog(dir, sync_mode="sync")` — always with synchronous fsync
3. Performs operations (append, batch, checkpoint, truncate)
4. Asserts invariants on returned sequence numbers and replayed records
5. Closes the WAL and lets the temp directory clean up
6. Prints pass/fail per test

The corruption test has an extra step: it closes the WAL, locates the `.wal` file via glob, writes garbage bytes at the end, then reopens the WAL and checks that replay degrades gracefully.

## Invariants

These tests collectively enforce:

1. **Contiguous sequence numbering** — `append` returns 1, 2, 3, ... with no gaps. Batch operations consume one sequence number per operation in the batch.
2. **Checkpoint consumes a sequence number** — after 6 data records (3 individual + 3 batch), `current_seq_num()` is 7, and `checkpoint()` returns 8.
3. **Crash durability** — with `sync_mode="sync"`, records survive process death without `close()`.
4. **Corruption isolation** — corrupted data stops replay but does not discard valid preceding records.
5. **Truncation is exclusive** — `truncate(2)` removes records with seq <= 2, keeps seq 3+.
6. **Monotonic sequences across restarts** — sequence numbers never reset, even after close/reopen.
7. **`replay()` filters COMMIT records** — returns only data operations. `iterate()` includes everything.
8. **`replay(after_seq=n)` is exclusive** — returns records with seq > n.

## Error Handling

The tests themselves don't handle errors — assertion failures propagate as `AssertionError` and halt execution (since there's no test runner to catch them). The tests **verify** error handling in the WAL:

- Corruption in `test_corruption_stops_replay` expects the WAL to silently truncate replay at the corruption point rather than raising an exception.
- No explicit test for what happens when the directory doesn't exist, disk is full, or permissions are wrong — these are crash-path tests, not adversarial-input tests.

## Topics to Explore

- [file] `write-ahead-log/wal.py` — The WAL implementation itself: record format, CRC checksumming, fsync strategy, and how segment rotation works
- [function] `write-ahead-log/wal.py:replay` — How replay distinguishes valid from corrupted records, and whether it uses CRC or length-prefix validation
- [function] `write-ahead-log/wal.py:append_batch` — How batch atomicity is implemented — whether it uses a single write or COMMIT markers, and how sequence numbers are assigned within a batch
- [file] `write-ahead-log/test_wal.py` — The pytest-based test suite: likely overlaps with this file but may cover additional edge cases or use fixtures
- [general] `tester-vs-test-convention` — Why the repo maintains both `test_*.py` (pytest) and `tester_test_*.py` (standalone) files across all implementations

## Beliefs

- `wal-checkpoint-consumes-sequence` — `checkpoint()` increments the sequence counter by 1, so it occupies a position in the sequence number space alongside data records
- `wal-replay-excludes-commit-markers` — `replay()` filters out COMMIT records while `iterate()` returns all raw records including COMMIT
- `wal-corruption-truncates-not-raises` — When the WAL encounters a corrupted record during replay, it stops and returns all valid records before the corruption point rather than raising an exception
- `wal-batch-appends-three-ops-plus-commit` — `append_batch` with 3 operations produces 4 raw records (3 data + 1 COMMIT), with `iterate()` seeing all 4 and `replay()` seeing only the 3 data records
- `wal-sync-mode-survives-crash` — With `sync_mode="sync"`, records are durable immediately after `append` returns, even if the WAL is never closed

