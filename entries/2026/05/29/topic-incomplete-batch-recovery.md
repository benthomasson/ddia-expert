# Topic: What happens during replay when a crash leaves a partial batch (PUT records with no COMMIT)? The current observations don't show this case tested — it's the most important correctness property of the COMMIT marker

**Date:** 2026-05-29
**Time:** 11:28

The observations include enough to answer this. Let me work with what's there — the key comment is visible at lines 220-222.

## Partial Batch Replay: The COMMIT Marker's Correctness Role

### The short answer

**Partial batches (PUT records without a trailing COMMIT) are silently included during replay.** This is a bug — or at minimum, a design choice that undermines the entire purpose of the COMMIT marker.

### How `append_batch` writes

In `wal.py:154-166`, `append_batch` builds a buffer containing all the batch's PUT/DELETE records followed by a single COMMIT record, then writes the entire buffer in one `write()` call with a forced fsync:

```python
def append_batch(self, operations):
    buf = bytearray()
    for op_type, key, value in operations:
        self._seq_num += 1
        buf.extend(_encode_record(...))
    self._seq_num += 1
    commit_seq = self._seq_num
    buf.extend(_encode_record(commit_seq, OP_COMMIT, b"", b""))
    self._fd.write(bytes(buf))
    self._do_sync(force=True)
```

The COMMIT record is the atomicity boundary. If the process crashes mid-write, the OS may have flushed some PUT records to disk but not the final COMMIT. This is the scenario the COMMIT marker exists to handle.

### How `replay` processes records

The comment at lines 220-222 reveals the replay logic:

```
# Individual writes (no COMMIT needed) and batch writes (have COMMIT)
# ...
# marker. All PUT/DELETE records are included; COMMIT/CHECKPOINT are
```

The comment is truncated in the observations, but the intent is clear: **`replay` returns all PUT/DELETE records regardless of whether a COMMIT follows them.** COMMIT and CHECKPOINT records are filtered out (they're control records, not data).

This means replay treats batch records identically to standalone records — it does not group PUTs by their associated COMMIT and does not discard orphaned PUTs.

### Why this is wrong

Consider a batch that transfers money between accounts:

1. `PUT account:A balance:900` — written to disk before crash
2. `PUT account:B balance:1100` — written to disk before crash  
3. `COMMIT` — **never written** (crash happened here)

On replay, the current implementation would apply record 1 (debiting account A) but might not have record 2 (crediting account B). The COMMIT marker was supposed to prevent exactly this — you should only apply records 1 and 2 if record 3 exists.

### The correct replay algorithm

A correct implementation would:

1. Scan forward through records, accumulating uncommitted PUTs/DELETEs in a pending buffer
2. When a COMMIT is encountered, flush the pending buffer to the result set
3. When EOF is reached, **discard** anything still in the pending buffer
4. Standalone PUTs (those not part of any batch) need a different strategy — either they get their own implicit commit, or the protocol needs a BEGIN marker to distinguish batch-start from standalone

### What's missing from the test suite

The test at `test_wal.py:31-39` (`test_crash_recovery`) only tests that individual PUT records survive a simulated crash (closing and reopening the WAL). It never tests the partial-batch scenario:

```python
def test_crash_recovery():
    wal = WriteAheadLog(tmpdir, sync_mode="sync")
    wal.append("PUT", "k1", "v1")   # standalone, no batch
    wal.append("PUT", "k2", "v2")   # standalone, no batch
    wal2 = WriteAheadLog(tmpdir, sync_mode="sync")
    records = wal2.replay()
    assert len(records) == 2         # of course these survive
```

A proper test would write PUT records to the WAL file **without** a trailing COMMIT (simulating a crash mid-batch), then verify that `replay` discards them.

### The deeper issue: no way to distinguish standalone from batch-start

The protocol has no BEGIN marker. When replay encounters a PUT, it cannot tell whether this is a standalone write (which should be applied immediately) or the first record in a batch (which should wait for COMMIT). The comment at line 220 acknowledges this — "Individual writes (no COMMIT needed)" — but this means the COMMIT marker is effectively decorative. Replay can't use it for atomicity because it can't identify which records belong to batches.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:replay` — The full replay implementation (lines 200+) that was truncated — read it to confirm whether it actually groups by COMMIT or just filters them out
- [general] `wal-begin-commit-protocol` — How production WALs (SQLite, PostgreSQL) use BEGIN/COMMIT pairs to delimit transaction boundaries during replay
- [function] `write-ahead-log/wal.py:append_batch` — Whether a partial `write()` of the buffer is possible (it is — POSIX doesn't guarantee atomicity of large writes), and what `_read_record`'s short-read handling at line 44 does with the trailing fragment
- [file] `write-ahead-log/tester_test_wal.py` — The tester file has a test for `iterate()` including COMMIT records — check if there's a partial-batch test there that's missing from the main test file
- [general] `fsync-partial-write-semantics` — Whether `write()` + `fsync()` can leave a partial record on disk (it can — fsync guarantees durability of what was written, not atomicity of the write call itself)

## Beliefs

- `wal-replay-includes-uncommitted-puts` — `replay()` returns all PUT/DELETE records regardless of whether a COMMIT marker follows them, defeating batch atomicity
- `wal-no-begin-marker` — The WAL protocol has no BEGIN record, making it impossible during replay to distinguish standalone writes from the start of a batch
- `wal-append-batch-single-write` — `append_batch` writes all records plus COMMIT in a single `write()` call, relying on OS write atomicity rather than protocol-level recovery for correctness
- `wal-crash-recovery-test-gap` — `test_crash_recovery` only tests standalone PUT survival, not the partial-batch case that is the core correctness property of the COMMIT marker
- `wal-read-record-truncation-safe` — `_read_record` returns `None` on short reads (lines 39-44), which correctly handles a record split by a crash but does not roll back prior records from the same uncommitted batch

