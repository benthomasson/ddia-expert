# Topic: Explore what happens if `append_batch` writes a buffer that pushes past `max_file_size` — does the batch stay atomic or can it split?

**Date:** 2026-05-29
**Time:** 08:09

## Batch Atomicity Across File Rotation in the WAL

**The batch stays atomic. It can never split across files.**

The key is the ordering of operations inside `append_batch` at `write-ahead-log/wal.py:153`:

```python
def append_batch(self, operations: List[Tuple[str, str, str]]) -> int:
    with self._lock:
        buf = bytearray()
        for op_type, key, value in operations:
            self._seq_num += 1
            buf.extend(_encode_record(...))
        self._seq_num += 1
        commit_seq = self._seq_num
        buf.extend(_encode_record(commit_seq, OP_COMMIT, b"", b""))
        self._fd.write(bytes(buf))          # 1. write entire batch
        self._do_sync(force=True)           # 2. fsync
        self._maybe_rotate()               # 3. THEN check rotation
        return commit_seq
```

Three things guarantee atomicity:

### 1. Single buffer, single write

All operation records plus the COMMIT marker are assembled into one `bytearray`, then flushed with a single `self._fd.write(bytes(buf))`. There's no per-record rotation check inside the loop. The buffer can be arbitrarily large — it doesn't matter.

### 2. Rotation is post-write

`_maybe_rotate()` (line 136) runs **after** the write and sync complete:

```python
def _maybe_rotate(self):
    if self._fd and self._fd.tell() >= self._max_file_size:
        self._rotate()
```

This means a WAL file **can grow beyond `max_file_size`** if a batch pushes it past the limit. The size cap is soft, not hard. The file only rotates on the *next* operation after the batch finishes.

### 3. The lock prevents interleaving

The entire method runs under `self._lock`, so no other thread can sneak a rotation or a competing write between the batch records.

### Contrast with single-record `append`

The single-record `append` at line 141 follows the same write-then-rotate pattern, but for individual records it's less consequential — each record is self-contained. The batch case is where this ordering decision actually matters for correctness.

### Contrast with Bitcask

Bitcask (`hash-index-storage/bitcask.py:169`) calls `_maybe_rotate()` **before** each `put`, which means consecutive puts can land in different files. That's fine for Bitcask since it doesn't need batch semantics — each key-value pair is independently addressable via the in-memory `keydir`.

### What could go wrong

If the process crashes *during* `self._fd.write(bytes(buf))` — after some bytes hit disk but before the COMMIT record is written — recovery code should treat the batch as incomplete. The COMMIT record (`OP_COMMIT`) at the end of the buffer serves as the durability marker: if it's absent, the batch is discarded during replay. This is the standard WAL pattern from DDIA Chapter 3.

### One subtle implication

Because `_open_latest` (line 100) checks `os.path.getsize(last) < self._max_file_size` on startup, a file that was pushed past the limit by a batch will trigger a new file on the next open. The overshoot is tolerated for exactly one batch's worth.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_record` — How recovery handles partial/corrupt records at end-of-file, which is exactly what you'd see after a crash mid-batch
- [function] `write-ahead-log/wal.py:truncate` — How committed batches are garbage-collected and whether truncation respects batch boundaries
- [file] `write-ahead-log/test_wal.py` — The test at line 75 sets `max_file_size=100`, which is small enough to force rotation — see if there's a test covering batch-across-rotation
- [function] `hash-index-storage/bitcask.py:_maybe_rotate` — Compare the pre-write rotation strategy to the WAL's post-write strategy and why each design fits its use case
- [general] `commit-record-replay-semantics` — How the replay loop uses OP_COMMIT to distinguish complete batches from partial writes during crash recovery

## Beliefs

- `wal-batch-never-splits` — `append_batch` writes all records plus COMMIT in a single `fd.write()` call; rotation only occurs after the write completes, so a batch can never span two WAL files
- `wal-max-file-size-is-soft` — `max_file_size` is a soft limit; a single batch can push a WAL file arbitrarily past it because `_maybe_rotate` runs post-write
- `wal-batch-forces-fsync` — `append_batch` calls `_do_sync(force=True)`, bypassing the batch sync counter and always issuing an fsync regardless of sync mode
- `wal-commit-record-is-atomicity-marker` — The `OP_COMMIT` record appended at the end of a batch buffer is what distinguishes a complete batch from a partial write during crash recovery

