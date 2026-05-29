# Function: WAL in log-structured-merge-tree/lsm.py

**Date:** 2026-05-29
**Time:** 08:09

# `WAL` — Write-Ahead Log for Crash Recovery

## 1. Purpose

`WAL` is a write-ahead log that ensures durability of writes to the LSM tree's in-memory memtable. Every mutation (put or delete) is appended to the WAL *before* it's applied to the memtable. If the process crashes, the memtable (which lives only in RAM) is lost — but the WAL file on disk contains every operation since the last flush. On restart, `replay()` reconstructs the memtable from the WAL, so no acknowledged writes are lost.

This is the classic WAL pattern from DDIA Chapter 3: the log turns an in-memory structure into a durable one without paying the cost of writing a full sorted structure on every operation.

## 2. Contract

- **Precondition**: The directory containing `path` must exist before construction.
- **Postcondition (append)**: After `append()` returns, the key-value pair is durable on disk (modulo OS/hardware buffering — `flush()` is called but not `fsync()`).
- **Postcondition (replay)**: Returns all complete entries written since the last `truncate()`. Partial entries from a mid-write crash are silently skipped.
- **Postcondition (truncate)**: The WAL file is emptied. The file descriptor is reopened for future appends.
- **Invariant**: The WAL contains a contiguous sequence of length-prefixed key-value records with no framing or checksums between them.

## 3. Parameters

### `__init__(self, path: str)`
- `path` — Absolute or relative filesystem path to the WAL file. Opened in append-binary mode (`"ab"`), so it is created if absent and existing content is preserved.

### `append(self, key: str, value: bytes)`
- `key` — A UTF-8 string. Encoded to bytes internally. No length limit is enforced, but keys larger than ~4 GiB would overflow the 4-byte length prefix.
- `value` — Raw bytes. For deletes, the caller passes `TOMBSTONE` (`b""`), which is a zero-length value. Same 4 GiB implicit limit applies.

### `replay(self) -> List[Tuple[str, bytes]]`
No parameters. Returns all recoverable entries from the file at `self._path`.

## 4. Return Value

- `append` — `None`. Fire-and-forget; the caller's obligation is just to call it before mutating the memtable.
- `replay` — `List[Tuple[str, bytes]]`: a list of `(key, value_bytes)` pairs in insertion order. An empty list if the file doesn't exist or contains no complete records. The caller (`LSMTree._replay_wal`) feeds these directly into the memtable via `self._memtable[key] = value`.
- `truncate` — `None`.
- `close` — `None`.

## 5. Algorithm

### Binary Record Format

Each record is 4 fields, no separators:

```
[key_len: 4 bytes big-endian uint32]
[key: key_len bytes, UTF-8]
[value_len: 4 bytes big-endian uint32]
[value: value_len bytes]
```

### `append`

1. Encode the key to UTF-8 bytes.
2. Write the key length as a 4-byte big-endian unsigned integer (`>I`).
3. Write the key bytes.
4. Write the value length as a 4-byte big-endian unsigned integer.
5. Write the value bytes.
6. Flush the userspace buffer to the OS. This does **not** call `fsync()`, so durability depends on OS write-back behavior.

### `replay`

1. If the file doesn't exist, return empty.
2. Read the entire file into memory (`data`).
3. Walk through `data` with a manual cursor (`pos`), parsing each record:
   - Read 4 bytes → `klen`. If fewer than 4 bytes remain, stop (partial record from crash).
   - Read `klen` bytes → key. If fewer than `klen` remain, stop.
   - Read 4 bytes → `vlen`. If fewer than 4 bytes remain, stop.
   - Read `vlen` bytes → value. If fewer than `vlen` remain, stop.
   - Append `(key, value)` to the result list.
4. Return all successfully parsed entries.

The boundary checks at every step make this crash-tolerant: if the process died mid-write, the incomplete trailing record is simply ignored.

### `truncate`

1. Close the current file descriptor.
2. Reopen the file in write-binary mode (`"wb"`), which truncates it to zero bytes.
3. Close that descriptor.
4. Reopen in append-binary mode (`"ab"`) for future writes.

This three-step dance is necessary because `"wb"` truncates on open, but subsequent appends need `"ab"` mode.

## 6. Side Effects

- **Disk I/O**: `append` writes to and flushes the WAL file on every call. `replay` reads the entire file. `truncate` rewrites the file.
- **File descriptor lifecycle**: The object holds an open file descriptor (`self._fd`) for its entire lifetime. `truncate` closes and reopens it. `close` closes it permanently.
- **No fsync**: `flush()` pushes data from Python's buffer to the OS, but does not force it to stable storage. A power failure (not just a process crash) could lose the last few appended records.

## 7. Error Handling

There is essentially **no error handling**. Specifically:

- If `path`'s parent directory doesn't exist, `__init__` raises `FileNotFoundError`.
- If the disk is full, `append` raises `OSError` mid-write, potentially leaving a partial record. The `replay` logic tolerates this by design.
- `replay` opens a separate file descriptor (not `self._fd`), so it works even if the WAL was opened for appending by another instance — though this class isn't designed for concurrent access.
- No checksums are written or verified. A bitflip in the length prefix could cause `replay` to parse garbage or skip records silently. There's no CRC or magic-number framing.

## 8. Usage Patterns

Within `LSMTree`:

```python
# On startup — recover memtable from WAL
self._wal = WAL(os.path.join(data_dir, "wal.log"))
self._replay_wal()  # calls self._wal.replay()

# On every put/delete — log before memtable mutation
self._wal.append(key, v)
self._memtable[key] = v

# On flush — memtable becomes SSTable, WAL is reset
self._wal.truncate()

# On shutdown
self._wal.close()
```

The caller (`LSMTree`) is responsible for:
1. Calling `append` **before** mutating the memtable (write-ahead invariant).
2. Calling `truncate` **after** the SSTable is fully written (otherwise data is lost).
3. Calling `close` on shutdown.

## 9. Dependencies

- `struct` (stdlib) — binary packing/unpacking with big-endian format.
- `os` (stdlib) — `os.path.exists` in `replay`.
- No external dependencies. This is a self-contained, minimal implementation.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:LSMTree._flush` — The critical caller: see how the WAL truncate is sequenced relative to SSTable write, and what happens if the process crashes between them.
- [function] `log-structured-merge-tree/lsm.py:LSMTree._replay_wal` — Shows how replayed entries are fed back into the memtable, and why replay order (insertion order) produces correct final state via last-write-wins.
- [file] `write-ahead-log/test_wal.py` — Likely contains crash-recovery test scenarios and edge cases that document the behavioral contract more precisely than the code alone.
- [general] `wal-fsync-durability` — This WAL calls `flush()` but not `os.fsync()`. Explore the difference: `flush` guarantees process-crash durability but not power-failure durability. Real databases (Postgres, SQLite) always fsync the WAL.
- [general] `wal-checksum-framing` — Production WALs (e.g., RocksDB, LevelDB) include CRC32 checksums per record to detect corruption, not just truncation. This implementation silently accepts corrupted records as long as the length fields parse.

## Beliefs

- `wal-no-fsync` — WAL calls `flush()` but never `os.fsync()`, so acknowledged writes can be lost on power failure (not just process crash).
- `wal-crash-tolerant-replay` — `replay()` silently discards any trailing partial record, making it tolerant of process crashes mid-`append`.
- `wal-no-checksums` — The WAL format has no checksums or magic bytes; it can only detect truncation, not corruption of existing bytes.
- `wal-truncate-after-flush` — `LSMTree._flush` truncates the WAL only after the SSTable is fully written, maintaining the invariant that no acknowledged write exists solely in volatile memory.
- `wal-append-before-memtable` — `LSMTree.put` and `LSMTree.delete` always call `WAL.append` before modifying the memtable, upholding the write-ahead invariant.

