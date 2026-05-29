# Function: commit in b-tree-storage-engine/btree.py

**Date:** 2026-05-28
**Time:** 18:52

## `WAL.commit` — Finalize a WAL transaction by clearing the log

### Purpose

`commit` marks the end of a write-ahead log transaction. After all dirty pages have been flushed to the data file, this method clears the WAL, signaling that the writes are durable and no recovery is needed on restart. It implements the "checkpoint" step of WAL-based crash safety: once the data file is known-good, the log entries that got it there are no longer needed.

### Contract

- **Precondition**: All page writes logged via `log_write` have already been applied to the `PageManager` (the data file contains the correct page images). The caller is responsible for ensuring logical consistency of the written pages before calling commit.
- **Postcondition**: The WAL file is empty and `fsync`'d — a crash after commit returns will not trigger any replay. The sequence counter is reset to 0, ready for the next batch of writes.
- **Invariant**: Between `log_write` calls and `commit`, the WAL contains a recoverable record of all in-flight page mutations. After `commit`, the WAL contains nothing.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `page_manager` | `PageManager` | The page I/O layer whose data file must be synced before the WAL is cleared. |

### Return Value

`None`. This is a side-effect-only method.

### Algorithm

1. **`page_manager.sync()`** — Flush the data file's OS buffer and `fsync` it. This guarantees every page written via `PageManager.write_page` is durable on disk, not just in kernel buffers. This step *must* happen before the WAL is cleared — otherwise a crash could leave both the data file incomplete and the WAL empty, causing data loss.

2. **`self._f.seek(0)` + `self._f.truncate(0)`** — Erase the WAL file contents entirely. `seek(0)` positions the file cursor at the start; `truncate(0)` discards all bytes. After this, the WAL is logically empty.

3. **`self._f.flush()` + `os.fsync(self._f.fileno())`** — Force the truncated (empty) WAL state to durable storage. Without this, the OS could still have the old WAL content cached, and a crash could "resurrect" stale log entries that would be replayed against already-committed data — corrupting it via double-application.

4. **`self._seq = 0`** — Reset the monotonic sequence counter so the next transaction's log entries start from 1.

### Side Effects

- **Disk I/O on the data file**: `page_manager.sync()` triggers `flush` + `fsync` on `btree.dat`.
- **Disk I/O on the WAL file**: The WAL (`btree.wal`) is truncated to zero bytes and that truncation is `fsync`'d.
- **Internal state mutation**: `self._seq` is reset.

### Error Handling

No exceptions are caught. If `sync()`, `truncate()`, or `fsync()` fails (e.g., disk full, I/O error), the exception propagates to the caller. This is the correct behavior — a failed commit should not silently succeed, since the WAL may still be needed for recovery.

### Usage Patterns

`commit` is called at the end of every mutating B-tree operation (`put`, `delete`, `close`):

```python
# In BTree.put, after all page writes are done:
self.wal.commit(self.pm)
```

The pattern is always: (1) log page writes via `_wal_write_page` / `_wal_write_meta`, (2) call `commit` to finalize. The caller must not interleave unrelated writes between `log_write` and `commit`, since `commit` clears *all* log entries.

### Dependencies

- `PageManager.sync()` — relied on to make the data file durable before the WAL is cleared.
- `os.fsync` — POSIX system call for forcing kernel buffers to physical storage.
- The WAL file handle `self._f` — opened in `__init__`, must remain valid.

### Key Assumption

The ordering guarantee — **data file fsync before WAL truncation fsync** — is what makes crash recovery correct. The code assumes that `os.fsync` provides the barrier semantics described by POSIX: after `fsync` returns, the data is on stable storage. On some hardware (e.g., drives with volatile write caches that lie about flush completion), this assumption can be violated. The code does not use `O_DIRECT` or `fdatasync`; it trusts the OS and hardware to honor `fsync`.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:WAL.recover` — The counterpart to `commit`: replays logged pages after a crash, then clears the WAL — understanding both sides reveals the full crash-safety contract
- [function] `b-tree-storage-engine/btree.py:WAL.log_write` — How individual page mutations are serialized with sequence numbers and CRC checksums before `commit` finalizes them
- [function] `b-tree-storage-engine/btree.py:BTree._wal_write_page` — The caller-side pattern that pairs WAL logging with page writes, showing how the WAL wraps every mutation
- [general] `write-ahead-logging-fsync-ordering` — The ARIES-style invariant that the log must be durable before the data file changes are considered committed, and how this code inverts that (data durable → log cleared) because it uses a redo-only WAL
- [file] `write-ahead-log/test_wal.py` — Tests for WAL crash-safety scenarios that validate the commit/recover contract under simulated failures

---

## Beliefs

- `wal-commit-sync-before-truncate` — `WAL.commit` always fsyncs the data file before truncating the WAL; reversing this order would create a crash-safety hole where committed data could be lost
- `wal-commit-clears-all-entries` — `commit` clears the entire WAL unconditionally; there is no partial commit or transaction grouping within a single WAL file
- `wal-seq-reset-on-commit` — The WAL sequence counter resets to 0 on every commit, meaning sequence numbers are only meaningful within a single uncommitted transaction window
- `wal-is-redo-only` — This WAL uses redo-only recovery (replay logged pages forward); there is no undo log, so a crash mid-transaction before commit means the incomplete writes in the data file may be partially applied but the WAL replay will overwrite them to a consistent state

