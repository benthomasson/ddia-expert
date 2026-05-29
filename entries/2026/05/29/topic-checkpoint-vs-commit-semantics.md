# Topic: The WAL has both `OP_COMMIT` (for batch atomicity) and `OP_CHECKPOINT` (for truncation safety); understanding when each is used clarifies the two distinct durability guarantees

**Date:** 2026-05-29
**Time:** 08:05

## Two Durability Guarantees: Commit vs. Checkpoint

This WAL implementation provides two distinct safety markers that serve different purposes. They're easy to confuse because both are "just records in the log," but they protect against different failure modes.

### `OP_COMMIT` — Batch Atomicity

`OP_COMMIT` answers the question: **"Did all operations in this group make it to the log?"**

It's written exclusively by `append_batch()` (`wal.py:155–167`). The method buffers all individual operations plus a trailing COMMIT record, then writes the entire buffer in a single `self._fd.write(bytes(buf))` call followed by a forced fsync:

```python
# wal.py:155-167
def append_batch(self, operations: List[Tuple[str, str, str]]) -> int:
    with self._lock:
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

During replay, if the recovery code sees a PUT, PUT, DELETE sequence with no trailing COMMIT, it knows the batch was interrupted — those records are incomplete and should be discarded. The COMMIT record is the "all-or-nothing" boundary. The test at `test_wal.py:13–19` confirms this: a 3-operation batch produces a COMMIT at seq 7, and `iterate()` (`test_wal.py:88–92`) shows the COMMIT is present as a real record in the log.

Compare this to `append()` (`wal.py:147–154`), which writes individual records with no COMMIT. Single operations are their own atomic unit — they either made it to disk or they didn't. The COMMIT marker only matters when multiple operations must succeed or fail together.

### `OP_CHECKPOINT` — Truncation Safety

`OP_CHECKPOINT` answers a different question: **"Up to what point has the main data store absorbed these changes?"**

It's written by `checkpoint()` (`wal.py:169–176`) as a standalone record, also with a forced fsync:

```python
# wal.py:169-176
def checkpoint(self) -> int:
    with self._lock:
        self._seq_num += 1
        seq = self._seq_num
        self._fd.write(_encode_record(seq, OP_CHECKPOINT, b"", b""))
        self._do_sync(force=True)
        return seq
```

The checkpoint sequence number becomes the argument to `truncate()` (`wal.py:178+`), which removes all records at or below that sequence. The test at `test_wal.py:19–25` demonstrates the intended protocol:

```python
cp_seq = wal.checkpoint()       # "main store is consistent up to here"
records = wal.replay(after_seq=cp_seq)
assert len(records) == 0        # nothing left to replay
```

The caller is saying: "I've flushed everything up to this point to the main store. If I crash after this, I only need to replay records *after* the checkpoint." This is what makes `truncate(cp_seq)` safe — you're only discarding WAL entries that the main store already reflects.

### The Two Guarantees Together

| Marker | Protects against | Written by | Used during |
|--------|-----------------|------------|-------------|
| `OP_COMMIT` | Partial batch application | `append_batch()` | Recovery/replay |
| `OP_CHECKPOINT` | Premature WAL truncation | `checkpoint()` | Truncation |

A typical lifecycle: write batches (each sealed with COMMIT) → flush to main store → write CHECKPOINT → truncate WAL up to checkpoint. COMMIT ensures each batch is atomic within the log. CHECKPOINT ensures you don't delete log entries the main store hasn't absorbed yet.

Note that `replay()` filters out both COMMIT and CHECKPOINT records (returning only PUT/DELETE data operations), while `iterate()` returns everything including markers — the test at `test_wal.py:88–92` asserts `all_recs[3].op_type == "COMMIT"` to verify this.

## Topics to Explore

- [function] `write-ahead-log/wal.py:replay` — The replay method (past line 200, not shown) is where the COMMIT filtering logic lives; understanding how it handles incomplete batches is critical to the atomicity guarantee
- [function] `write-ahead-log/wal.py:truncate` — The truncation implementation at line 178+ shows how checkpoint sequences gate which WAL files get deleted vs. rewritten
- [general] `batch-sync-mode` — The `_do_sync` method (`wal.py:131–140`) has three sync modes (sync/batch/none) with different durability tradeoffs; `append_batch` and `checkpoint` both force sync regardless of mode, which is worth understanding
- [file] `write-ahead-log/tester_test_wal.py` — Contains `test_checkpoint_replay_after` (line 139) which tests the checkpoint-then-replay-after pattern more explicitly than the main test file
- [general] `incomplete-batch-recovery` — What happens during replay when a crash leaves a partial batch (PUT records with no COMMIT)? The current observations don't show this case tested — it's the most important correctness property of the COMMIT marker

## Beliefs

- `commit-only-in-batch` — `OP_COMMIT` is written exclusively by `append_batch()`; single `append()` calls never produce a COMMIT record
- `checkpoint-and-commit-force-sync` — Both `checkpoint()` and `append_batch()` call `_do_sync(force=True)`, bypassing the batch sync mode to guarantee immediate fsync
- `replay-filters-markers` — `replay()` returns only data operations (PUT/DELETE), filtering out COMMIT and CHECKPOINT records; `iterate()` returns all record types including markers
- `checkpoint-seq-gates-truncate` — The sequence number returned by `checkpoint()` is the intended argument to `truncate()`, establishing the safe truncation boundary
- `batch-write-is-single-call` — `append_batch()` assembles all operations plus COMMIT into a single `bytearray` and writes it in one `fd.write()` call, minimizing the window for partial writes

