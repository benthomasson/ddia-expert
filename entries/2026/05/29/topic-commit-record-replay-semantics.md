# Topic: How the replay loop uses OP_COMMIT to distinguish complete batches from partial writes during crash recovery

**Date:** 2026-05-29
**Time:** 11:44

# How the Replay Loop Uses OP_COMMIT During Crash Recovery

## The Short Version

**It doesn't.** The `replay()` method's docstring claims it "skips uncommitted batches" (`wal.py:215`), but the implementation returns all valid PUT/DELETE records regardless of whether a COMMIT record follows them. The COMMIT marker exists in the protocol but is filtered out during replay, not used as a batch boundary.

## The Write Side: How Batches Are Committed

`append_batch` (`wal.py:153-167`) builds a single `bytearray` containing every operation's encoded record, then appends an `OP_COMMIT` record (value `3`, defined at `wal.py:12`) at the end:

```
seq 4: PUT order:1 item_a    ← batch member
seq 5: PUT order:2 item_b    ← batch member
seq 6: DELETE order:0         ← batch member
seq 7: COMMIT                 ← batch boundary
```

The entire buffer — all operations plus the trailing COMMIT — is written in a single `self._fd.write(bytes(buf))` call (`wal.py:164`), followed by a forced `fsync` (`wal.py:165`). This is where atomicity actually lives: either the OS writes the whole buffer to disk or it doesn't. The COMMIT record is part of the payload, not a separate step.

## The Read Side: How Replay Handles It

The replay implementation (`wal.py:212+`) processes records through `_read_all_records`, which calls `_read_record` (`wal.py:38-57`) for each entry. The inline comment at `wal.py:220-222` is explicit:

> Individual writes (no COMMIT needed) and batch writes (have COMMIT) are indistinguishable in the log format since there's no batch-start marker.

Replay filters records by type — keeping only PUT and DELETE, discarding COMMIT and CHECKPOINT — and by sequence number (the `after_seq` parameter). It does **not** scan ahead for a COMMIT before including a PUT, nor does it buffer records pending a COMMIT boundary.

## How Crash Recovery Actually Works

The atomicity guarantee comes from two layers below replay:

1. **`_read_record` truncation detection** (`wal.py:43-44`): If a record is partially written (crash mid-write), the length header or the record body will be short. `_read_record` returns `None`, ending iteration for that file.

2. **CRC verification** (`wal.py:52-55`): If bytes landed on disk but were garbled (sector tearing, partial page flush), the CRC check fails and `_read_record` raises `ValueError`. `_read_all_records` catches this and **stops iteration entirely** — all records after the corruption point are discarded.

So if a crash happens mid-batch:
- Some PUT records may have reached disk, but the trailing COMMIT (and possibly some PUTs) will be truncated or corrupt
- `_read_record` detects the damage at the truncation/corruption point
- Replay halts, discarding everything from that point forward
- The surviving PUT records from earlier in the batch **are returned** by replay

This means partial batch records can survive replay if the crash happened after some records were individually fsynced but before the whole buffer completed. In practice, because `append_batch` writes the entire buffer in one `write()` call, the window for this is narrow — but POSIX doesn't guarantee atomicity of large writes, so it's not zero.

## The Gap

The protocol has no `OP_BEGIN` or `OP_BATCH_START` marker (`wal.py:10-14` defines only PUT, DELETE, COMMIT, CHECKPOINT). Without a begin marker, replay cannot distinguish "this PUT is standalone and should be applied" from "this PUT belongs to a batch that hasn't committed yet." The comment at `wal.py:221` acknowledges this directly.

The test suite confirms the gap: `test_crash_recovery` (`test_wal.py:31-39`) only verifies that standalone PUTs survive a reopen — it never tests the partial-batch scenario where PUTs exist on disk without a trailing COMMIT.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_all_records` — The corruption-halting iterator that replay delegates to; its early-termination is the real crash recovery mechanism
- [function] `log-structured-merge-tree/lsm.py:_replay_wal` — How the LSM tree consumes replay output to rebuild its memtable, and whether it adds its own batch-aware filtering
- [function] `write-ahead-log/wal.py:iterate` — Returns all raw records including COMMIT/CHECKPOINT, unlike replay's filtered view — a consumer could use this to build correct batch-aware recovery
- [general] `wal-begin-commit-protocol` — How production WALs (PostgreSQL, SQLite) use explicit BEGIN/COMMIT pairs to delimit transactions during replay, which this implementation lacks
- [general] `fsync-partial-write-semantics` — Whether a single large `write()` + `fsync()` can leave partial records on disk (it can), and what that means for the single-buffer atomicity assumption

## Beliefs

- `wal-replay-ignores-commit-markers` — `replay()` filters out OP_COMMIT records as control markers rather than using them as batch boundaries; all valid PUT/DELETE records are returned regardless of whether a COMMIT follows
- `wal-crash-safety-via-truncation-detection` — Crash recovery relies on `_read_record` detecting truncated or CRC-invalid records and halting iteration, not on COMMIT-aware batch grouping
- `wal-no-begin-marker-prevents-batch-grouping` — The WAL format defines no batch-start opcode, making it structurally impossible for replay to distinguish standalone writes from uncommitted batch members
- `wal-docstring-implementation-mismatch` — The replay docstring claims "skips uncommitted batches" but the implementation returns all valid PUT/DELETE records, a documented-but-unimplemented invariant

