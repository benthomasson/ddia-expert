# Topic: The WAL stops at the first corrupted record rather than scanning past it; explore whether this is always safe (what if corruption hits the middle of a multi-file WAL?)

**Date:** 2026-05-29
**Time:** 06:26

# WAL Stop-at-First-Corruption: Is It Always Safe?

## The Core Mechanism

The stop-at-first-corruption strategy is implemented in `_read_record` (`wal.py:37-58`). It has two exit paths:

1. **EOF or partial read** → returns `None` (benign, normal end-of-file)
2. **CRC mismatch** → raises `ValueError` (corruption detected)

Every caller handles this the same way — `break` out of the read loop:

```python
# _recover_seq_num, wal.py:85-96
for path in self._wal_files():
    with open(path, "rb") as f:
        while True:
            try:
                rec = _read_record(f)
                if rec is None:
                    break
                max_seq = max(max_seq, rec.seq_num)
            except ValueError:
                break   # ← stops reading THIS file
```

## The Multi-File Nuance

Here's the critical detail: **the behavior differs between same-file and cross-file corruption**.

In `_recover_seq_num`, the `break` exits the inner `while True` loop, but the outer `for path in self._wal_files()` loop continues. So if file 2 of 4 has corruption mid-way:

| File | Records | What happens |
|------|---------|-------------|
| `000001.wal` | seq 1–50 | All read ✓ |
| `000002.wal` | seq 51–100, corruption at seq 75 | Reads 51–74, stops. **Seq 75–100 lost** |
| `000003.wal` | seq 101–150 | All read ✓ |
| `000004.wal` | seq 151–200 | All read ✓ |

The recovered `max_seq` would be 200 (correct), but `replay` (whose source is past line 200, not included in the observations) likely follows the same pattern: it would **silently drop records 75–100** from file 2 while still replaying files 3 and 4.

## Where This Gets Dangerous

### 1. Lost records without any signal

Records 75–100 in the example above are valid, intact data. They're dropped solely because an earlier record in the same file had a bad CRC. The caller has no way to know these records existed. The `test_corruption` test (`test_wal.py:45-57`) only validates single-file corruption where the damaged record is at the tail — it never tests mid-file corruption with valid records following.

### 2. Torn batches across rotation boundaries

`append_batch` (`wal.py:153-168`) writes all operations plus a COMMIT record as one buffer, but `_maybe_rotate` (`wal.py:132-134`) runs *after* the write. If a batch is the last thing written before a crash and the file was near `max_file_size`, the batch could straddle a rotation boundary if the implementation were slightly different. In the current code, rotation happens post-write so a single batch stays in one file — but this is a fragile invariant that depends on `_maybe_rotate` never being called mid-batch.

### 3. Sequence number gaps become invisible

If corruption causes records 75–100 to be dropped from file 2, but files 3 and 4 replay fine, the consumer sees a gap from seq 74 to seq 101. Nothing in the current code flags this gap. A consumer that assumes monotonically increasing sequence numbers *without gaps* would silently accept an incomplete log.

## Is It Safe?

**For tail corruption (torn writes at crash time): yes.** This is the common case. A crash mid-write produces a partial record at the end of the last file. `_read_record` returns `None` for short reads, and no valid data follows.

**For mid-file corruption (bit rot, media errors): no.** Valid records after the corruption point are silently discarded. In a multi-file WAL, this creates a hole in the sequence — data before and after the corrupted file segment is recovered, but the gap is never reported.

Production WALs (PostgreSQL, SQLite, RocksDB) handle this differently: they either use per-record framing that allows scanning past corruption, or they treat any corruption as a hard error requiring operator intervention. This implementation takes the simpler path, which is appropriate for a teaching codebase but would need hardening for production use.

## Topics to Explore

- [function] `write-ahead-log/wal.py:replay` — The replay method (past line 200) is the primary consumer of recovery; verify whether it also continues across files or stops globally at first corruption
- [function] `write-ahead-log/wal.py:truncate` — Truncation also uses the break-on-corruption pattern (visible at line 199); if corruption precedes the truncation point, records that should be kept may be silently dropped
- [function] `log-structured-merge-tree/lsm.py:WAL` — The LSM tree has its own simpler WAL class (line 13); compare its corruption handling to the standalone WAL module
- [general] `batch-atomicity-across-rotation` — Explore what happens if `append_batch` writes a buffer that pushes past `max_file_size` — does the batch stay atomic or can it split?
- [general] `gap-detection-during-replay` — Whether any consumer of `replay` validates sequence number continuity, which would surface silent data loss from mid-file corruption

## Beliefs

- `wal-corruption-stops-per-file` — `_recover_seq_num` stops at the first corrupted record within each file but continues to subsequent WAL files, so cross-file recovery is resilient but intra-file records after corruption are lost
- `wal-no-gap-detection` — Neither the WAL reader nor any visible consumer checks for sequence number gaps after replay, making silent data loss from mid-file corruption undetectable
- `wal-batch-single-file` — `append_batch` writes the entire batch buffer in one `_fd.write` call before `_maybe_rotate`, so a batch never spans two WAL files under normal operation
- `wal-tail-corruption-safe` — Tail corruption (partial writes from crashes) is safely handled because `_read_record` returns `None` on short reads, and no valid records can follow a torn write at EOF

