# Topic: What data loss scenarios are possible given that this implementation has no write-ahead log and no fsync calls

**Date:** 2026-05-29
**Time:** 08:37

# Data Loss Scenarios: Missing fsync and WAL Gaps

The picture is more nuanced than "no WAL and no fsync" — some implementations have both, some have one, and some have neither. The gaps create distinct failure modes.

## The Vulnerable Implementations

### 1. LSM Tree WAL: flush without fsync

`log-structured-merge-tree/lsm.py:26` — the WAL's `append` method calls `self._fd.flush()` but never `os.fsync()`. This is the most dangerous pattern in the codebase because it *looks* safe but isn't.

`flush()` moves data from Python's userspace buffer into the OS kernel's page cache. It does **not** force the kernel to write to the physical disk. On power loss or kernel panic, everything in the page cache is gone. A user calls `put()`, the WAL writes the entry (line 242), `flush()` succeeds, and then the machine loses power — the WAL entry never made it to disk. On recovery, `_replay_wal()` (line 230) replays what's on disk, and that write is silently missing.

**Scenario**: You insert 50 keys, the OS buffers them all, power fails. On restart the WAL replays 0 entries. The memtable is empty. Those 50 keys are gone with no indication anything was lost.

### 2. Log-structured hash table: no WAL, no fsync

`log-structured-hash-table/bitcask.py:155-157` — `_write_record()` calls `self._active_file.flush()` with no `os.fsync()`. This implementation has no WAL at all, which for an append-only log is fine by design (the data file *is* the log). But without fsync, the append-only property doesn't help:

```python
def _write_record(self, key: str, value: bytes) -> int:
    ...
    self._active_file.write(header + payload)
    self._active_file.flush()
    return offset
```

After `put()` returns (line 168), the in-memory index points to an offset that may not exist on disk yet. On crash and recovery, `_scan_segment()` reads up to the last complete record on disk. The index will be rebuilt without the lost records, but any caller who received a successful `put()` was told the write succeeded.

**Scenario**: Write key "order-123", get back a successful offset. Power fails. Restart calls `_recover()` (line 72), which scans segments and rebuilds the index. "order-123" is not on disk. `get("order-123")` returns `None`.

### 3. SSTable writer: no sync on finalize

`sstable-and-compaction/sstable.py:91-95` — `SSTableWriter.finish()` calls `self._f.close()` without any `fsync()` beforehand. Python's `close()` calls `flush()` implicitly but again, that's only to the page cache. If a crash occurs after `finish()` returns but before the OS flushes its buffers, you get a partially written SSTable on disk.

**Scenario**: LSM flush writes a new SSTable, truncates the WAL (line 314 of `lsm.py`), then power fails before the OS writes the SSTable to disk. On recovery, the WAL is empty (it was truncated) and the SSTable is corrupt or incomplete. Data that was in the memtable is lost permanently — it was removed from the WAL but never durably written to the SSTable.

### 4. Multi-step operations without atomic commit

The LSM `_flush()` method (line 303 of `lsm.py`) does at least three things: writes an SSTable, updates the SSTable list, and truncates the WAL. Without fsync between steps, a crash at any point leaves the system in an inconsistent state. The most dangerous ordering: WAL truncated first, SSTable not yet on disk.

## The Safe Implementations (for contrast)

- **B-Tree storage engine** (`b-tree-storage-engine/btree.py:107-108`): The `WAL.log_write()` calls both `flush()` and `os.fsync()`. The `PageManager.sync()` (line 100) and `close()` (line 104) also fsync. This is the correct pattern.
- **Hash index storage** (`hash-index-storage/bitcask.py:92-93`): Conditionally calls `os.fsync()` when `sync_writes=True`. Defaults to `True`.
- **Standalone WAL** (`write-ahead-log/wal.py:117-123`): Full fsync implementation with configurable sync modes ("sync", "batch", "none").

## Summary of Risk

| Implementation | WAL? | fsync? | Data loss window |
|---|---|---|---|
| B-Tree (`btree.py`) | Yes, with fsync | Yes | Minimal |
| Hash index (`hash-index-storage/bitcask.py`) | N/A (log is data) | Configurable | When `sync_writes=False` |
| LSM Tree (`lsm.py`) | Yes, **no fsync** | No | Any buffered write |
| Log-structured hash (`log-structured-hash-table/bitcask.py`) | None | No | Any buffered write |
| SSTable writer (`sstable.py`) | N/A | No | Flush-to-SSTable window |

The core problem isn't the absence of a WAL — it's that `flush()` without `fsync()` gives a false sense of durability. A developer reading `self._fd.flush()` might assume the data is safe. It isn't until `os.fsync(fd.fileno())` returns.

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:_flush` — The flush-to-SSTable path where WAL truncation can race against durable SSTable writes
- [function] `log-structured-hash-table/bitcask.py:_recover` — How index rebuild on startup silently drops records that never made it to disk
- [file] `write-ahead-log/wal.py` — The standalone WAL with proper fsync, batch sync, and rotation — the reference for how the other implementations should work
- [function] `b-tree-storage-engine/btree.py:WAL.commit` — Correct commit protocol: fsync the data file, then truncate the WAL — compare with the LSM tree's ordering
- [general] `fsync-vs-flush-semantics` — Understanding the OS page cache layer between `flush()` and physical disk writes, and how `fdatasync` differs from `fsync`

## Beliefs

- `lsm-wal-no-fsync` — The LSM Tree WAL (`lsm.py:26`) calls `flush()` but never `os.fsync()`, making all WAL entries vulnerable to loss on power failure despite appearing persisted
- `log-hash-table-no-durability-guarantee` — The log-structured hash table (`log-structured-hash-table/bitcask.py:155`) provides no durability guarantee: no WAL and no fsync means successful `put()` calls can silently lose data on crash
- `sstable-wal-truncation-race` — The LSM `_flush()` truncates the WAL after writing an SSTable but neither operation is fsynced, creating a window where both the WAL and SSTable data can be lost simultaneously
- `btree-wal-is-reference-implementation` — The B-Tree WAL (`btree.py:107-108`) is the only storage engine that correctly fsyncs both WAL entries and data pages, making it the only implementation with true crash safety
- `bitcask-sync-writes-default-true` — The hash-index Bitcask (`hash-index-storage/bitcask.py:33`) defaults `sync_writes=True` and conditionally fsyncs, making durability configurable — unlike the log-structured variant which has no fsync path at all

