# Topic: How the break-on-corruption strategy interacts with `flush()` guarantees and OS write atomicity — is a partial header possible, and what happens if `fsync` is not called?

**Date:** 2026-05-29
**Time:** 07:28

# Break-on-Corruption, `flush()` Guarantees, and Write Atomicity

## The Break-on-Corruption Strategy

The WAL uses a **stop-at-first-error** recovery strategy. Both `_recover_seq_num` (`wal.py:84-95`) and the reader functions (`_read_record` at `wal.py:37-58`) implement the same pattern:

1. Read the 4-byte length prefix. If fewer than 4 bytes are available, return `None` (EOF).
2. Read `record_length` bytes of payload. If fewer bytes are available, return `None` (partial record — treated as EOF, not an error).
3. Verify the CRC32 checksum. If it fails, **raise `ValueError`**.

Callers like `_recover_seq_num` (`wal.py:90-95`) and `replay`/`iterate` catch the `ValueError` and **break out of the loop**, discarding everything after the corrupt record — even if valid records follow it. This is a deliberate design choice: once corruption is detected, the log's sequential ordering guarantee is broken, so no subsequent record can be trusted.

## Is a Partial Header Possible?

**Yes, absolutely.** The record format written by `_encode_record` (`wal.py:29-34`) is:

```
[4B record_length][4B CRC][8B seq_num][1B op_type][4B key_len][key][4B val_len][value]
```

This is a multi-byte structure written as a single `self._fd.write(data)` call (`wal.py:155`). On POSIX systems, `write()` to a regular file is **not guaranteed to be atomic** for buffers larger than `PIPE_BUF` (typically 4096 bytes). Even for small writes, a crash can interrupt the operation at any byte boundary because:

1. **Python's `file.write()` goes through the C library's stdio buffer**, which may issue multiple `write(2)` syscalls.
2. **Even a single `write(2)` syscall only guarantees atomicity up to `PIPE_BUF` for pipes** — for regular files, the kernel can flush dirty pages to disk at any sub-record boundary.
3. **Without `fsync`, the OS page cache is the only buffer.** A power failure can lose any portion of a write that hasn't been flushed to the storage device, leaving a partial header on disk.

The `_read_record` function handles this: if fewer than 4 bytes are available for the length prefix (`wal.py:40-41`), or if the payload is shorter than `record_length` (`wal.py:43-44`), it returns `None`. This treats a partial header as a clean EOF — the record is simply lost, and recovery stops there.

However, there's a subtler scenario: **a partial write that overwrites the length field with a plausible but wrong value.** If a crash corrupts the 4-byte length prefix into a value that happens to be valid (e.g., it points to garbage data that still has 4+ bytes available), the reader would read `record_length` bytes of noise, compute a CRC, and almost certainly get a mismatch — triggering the `ValueError` at `wal.py:55-56`. The CRC acts as the safety net here.

## What Happens Without `fsync`?

The `_do_sync` method (`wal.py:123-134`) reveals three sync modes:

| Mode | Behavior | Durability |
|------|----------|------------|
| `"sync"` | `flush()` + `fsync()` on every append | Durable after `append()` returns |
| `"batch"` | `flush()` + `fsync()` every N writes | Last N-1 writes may be lost |
| `"none"` | Neither `flush()` nor `fsync()` | **No durability guarantee at all** |

The `"none"` mode (`wal.py:123-134` — note the implicit else: do nothing) is the dangerous case:

1. **`flush()` alone** pushes data from Python's userspace buffer to the OS kernel page cache. Without it, data sits in the Python process's memory — a SIGKILL loses it entirely, leaving the WAL file unchanged.

2. **Without `fsync()`**, even flushed data sits in the OS page cache. A power failure can lose it. The file on disk may contain:
   - Nothing new (entire write lost)
   - A partial record (kernel flushed some dirty pages but not all)
   - The complete record (kernel happened to flush everything)

In modes `"batch"` and `"none"`, the break-on-corruption strategy becomes the **only defense** against partial writes. The CRC check will catch garbage, and the short-read checks will catch truncation. But the consequence is **silent data loss**: records the application believed were committed are simply absent after recovery. The caller has no way to distinguish "this record was never written" from "this record was lost in a crash."

Notably, `append_batch` (`wal.py:159-170`) and `checkpoint` (`wal.py:172-178`) **force sync regardless of mode** (`force=True`), which means batch boundaries and checkpoint markers are always durable. Individual `append()` calls respect the configured mode, so in `"batch"` or `"none"` mode, the individual operations within a batch could theoretically be lost, but the batch COMMIT record itself is always synced. This creates an interesting asymmetry: you could lose individual puts but never a commit marker.

## The Interaction Summarized

The WAL's corruption strategy is **defensive but incomplete**:

- **CRC catches bit-rot and partial overwrites** — a half-written header with garbage bytes will fail the checksum.
- **Short-read checks catch truncation** — a crash mid-write that leaves fewer bytes than expected is treated as clean EOF.
- **Break-on-first-error is conservative** — it may discard valid records that follow a corrupt one, but it never replays corrupt data.
- **Without fsync, the whole mechanism shifts from "durability" to "best-effort consistency"** — you won't replay corrupt data, but you may silently lose records the application thought were committed.

The design faithfully mirrors what DDIA describes: the WAL is only as durable as the sync policy allows, and the CRC + break strategy ensures that whatever *is* on disk is either fully valid or cleanly discarded.

## Topics to Explore

- [function] `write-ahead-log/wal.py:append_batch` — Why batches force sync and how the COMMIT record creates an all-or-nothing recovery boundary
- [function] `write-ahead-log/wal.py:_read_record` — The exact byte-level parsing logic and how each failure mode (short read, CRC mismatch) maps to a different recovery outcome
- [file] `b-tree-storage-engine/btree.py` — Compare the WAL journal's checksum strategy (line 132-135) with how the B-tree handles write-ahead logging for page-level crash recovery
- [general] `write-ordering-and-barriers` — How the OS can reorder writes within a single `write()` call and why `fdatasync` vs `fsync` matters for metadata vs data durability
- [function] `hash-index-storage/bitcask.py:_write_record` — Bitcask uses the same `flush()` + conditional `fsync()` pattern (line 91-93) but has no CRC — compare the failure modes

## Beliefs

- `wal-break-on-corruption-discards-tail` — When `_read_record` raises `ValueError` (CRC mismatch), all callers break immediately, discarding any valid records that follow the corrupt one in the same file
- `wal-partial-header-treated-as-eof` — A short read of fewer than 4 bytes for the length prefix or fewer than `record_length` bytes for the payload returns `None`, indistinguishable from a normal end-of-file
- `wal-batch-commit-always-synced` — `append_batch` calls `_do_sync(force=True)`, ensuring the COMMIT record is fsynced regardless of the configured sync mode
- `wal-none-mode-no-flush-no-fsync` — In sync mode `"none"`, `_do_sync` is a no-op: data may remain in Python's userspace buffer and never reach disk before a crash
- `wal-crc-covers-content-not-header` — The CRC32 is computed over `op_type_byte + key + value` only; the length prefix, sequence number, and key length fields are not checksummed, meaning corruption in those fields is detected indirectly via downstream parse failures

