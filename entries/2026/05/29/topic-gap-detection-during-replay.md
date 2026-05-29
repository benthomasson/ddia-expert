# Topic: Whether any consumer of `replay` validates sequence number continuity, which would surface silent data loss from mid-file corruption

**Date:** 2026-05-29
**Time:** 08:10

## No Consumer Validates Sequence Number Continuity

**Short answer: No.** Neither `replay` itself nor any of its callers check that sequence numbers are contiguous. Mid-file corruption that destroys records cleanly (without leaving a partial/CRC-invalid record) would cause silent data loss.

### How `replay` handles corruption today

The WAL's `replay` method (`write-ahead-log/wal.py:212`) delegates to `_read_all_records` (`wal.py:233`), which reads records in a loop and **stops at the first corruption** — a CRC mismatch raises `ValueError`, caught by the caller, which breaks out of the loop. This means:

- **Tail corruption** (damaged bytes at the end of a file): detected, replay stops early, records after the damage are lost but the CRC error is the signal.
- **Mid-file corruption that damages a record's CRC**: detected — `_read_record` raises `ValueError` at `wal.py:55-56`. But replay **stops entirely**, silently dropping all valid records that follow the corrupted one.
- **Mid-file corruption that destroys a record completely** (e.g., zeroed-out bytes that cause `_read_record` to return `None` at `wal.py:41-42` due to a short read): replay silently stops. No error is raised. Records after the gap are lost.

### What's missing

`replay` returns a flat `List[WALRecord]` (`wal.py:212`). It filters by `after_seq` (`wal.py:226`) and skips uncommitted batches, but at no point does it verify that the returned sequence numbers form a contiguous sequence (e.g., `[3, 4, 5, 6]` rather than `[3, 6]`).

### The two callers

1. **LSM tree** (`log-structured-merge-tree/lsm.py:231`): calls `self._wal.replay()` and iterates over key-value pairs. It does not inspect `seq_num` at all — the LSM's own `replay` method (`lsm.py:28`) returns `List[Tuple[str, bytes]]`, stripping sequence numbers entirely.

2. **Tests** (`write-ahead-log/test_wal.py`): the corruption test at line 44 verifies that replay returns fewer records after corruption (`len(records) == 5`), but it only checks the count — it never asserts that the returned sequence numbers are consecutive.

### The gap

If records 3 and 4 (out of 1–6) are corrupted such that `_read_record` returns `None` or raises, replay stops at record 2. Records 5 and 6 are silently lost. No consumer would notice because:

- No code compares `len(replayed_records)` against the expected count derived from `current_seq_num()`.
- No code iterates the returned records checking `records[i+1].seq_num == records[i].seq_num + 1`.
- The LSM consumer discards sequence numbers before using the data.

A continuity check in `replay` or `_read_all_records` — something like flagging when the next record's `seq_num` jumps by more than 1 — would surface this class of silent data loss.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_all_records` — The low-level scan loop where corruption halts replay; understanding its `try/except ValueError: break` pattern is key to seeing why post-corruption records are silently dropped
- [function] `log-structured-merge-tree/lsm.py:replay` — The LSM's replay strips sequence numbers, making it impossible for downstream code to detect gaps
- [file] `write-ahead-log/test_wal.py` — Tests verify corruption *stops* replay but never assert continuity of the surviving records; a good place to add a gap-detection regression test
- [general] `multi-file-replay-ordering` — With WAL rotation, `_read_all_records` iterates files in sorted filename order; worth verifying that sequence numbers are monotonic across file boundaries, not just within a single file
- [function] `write-ahead-log/wal.py:_recover_seq_num` — Recovery scans for the *max* seq_num across all files but also stops at corruption (`wal.py:93-94`), meaning a corrupted early file could cause `_seq_num` to reset lower than actual, leading to duplicate sequence numbers on new writes

## Beliefs

- `wal-replay-stops-at-corruption` — `replay` stops reading at the first CRC mismatch or short read and silently discards all subsequent valid records in that file and later files
- `no-sequence-continuity-check` — No code in the codebase validates that replayed WAL records have contiguous sequence numbers; gaps from mid-file corruption go undetected
- `lsm-strips-seq-nums` — The LSM tree's `replay` method (`lsm.py:28`) returns `List[Tuple[str, bytes]]`, discarding sequence numbers and making gap detection impossible at the LSM layer
- `recover-seq-num-vulnerable-to-early-corruption` — `_recover_seq_num` (`wal.py:85`) breaks on `ValueError` during file scanning, so corruption in an early WAL file prevents it from discovering higher sequence numbers in later files

