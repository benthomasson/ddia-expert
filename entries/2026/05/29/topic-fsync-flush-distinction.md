# Topic: The difference between `file.flush()`, `os.fsync()`, and `os.fdatasync()` is foundational to understanding why Bug 1 and Bug 2 are durability violations, not just performance issues

**Date:** 2026-05-29
**Time:** 07:49

# `flush()` vs `fsync()`: Why Missing Syncs Are Durability Bugs

## The Three Layers of Write Buffering

When your Python program writes data, it passes through **three buffers** before reaching persistent storage:

```
Python buffer  →  OS page cache  →  Disk platter / SSD cells
     (1)              (2)                  (3)
```

Each of the three calls addresses exactly one boundary:

| Call | What it does | What survives a crash? |
|------|-------------|----------------------|
| `file.flush()` | Pushes data from Python's internal buffer (1) into the OS page cache (2) | **Nothing guaranteed** — the OS can still lose it |
| `os.fsync(fd)` | Forces the OS to write page cache (2) to physical media (3), including metadata (size, mtime) | **Data is on disk** |
| `os.fdatasync(fd)` | Same as `fsync` but skips metadata updates that aren't needed for reading the data back | **Data is on disk**, slightly faster |

The critical insight: **`flush()` alone gives you zero crash durability.** After `flush()`, your data sits in the OS page cache — kernel memory that vanishes on power loss, kernel panic, or OOM kill. The data *looks* written (another process can read it), but it isn't *durable*.

## Bug 1: LSM Tree WAL Never Calls `fsync`

In `log-structured-merge-tree/lsm.py`, the WAL's `append` method at **line 26**:

```python
def append(self, key: str, value: bytes):
    k = key.encode("utf-8")
    self._fd.write(struct.pack(">I", len(k)))
    self._fd.write(k)
    self._fd.write(struct.pack(">I", len(value)))
    self._fd.write(value)
    self._fd.flush()  # line 26 — pushes to OS, NOT to disk
```

There is no `os.fsync()` anywhere in this WAL class. The `truncate` method (line 59) closes and reopens the file, and `close` (line 64) just closes it — neither syncs.

**Why this is a durability violation, not a performance issue:** The entire *purpose* of a WAL is to guarantee that acknowledged writes survive crashes. If the machine loses power 1ms after `flush()` returns, every WAL entry still in the page cache is lost. The LSM tree's memtable was the only copy, and it's gone too. The user thinks their write was committed — it wasn't.

## Bug 2: Log-Structured Hash Table Never Calls `fsync`

In `log-structured-hash-table/bitcask.py`, the `_write_record` method (around **line 165**):

```python
def _write_record(self, key: str, value: bytes) -> int:
    key_bytes = key.encode("utf-8")
    payload = key_bytes + value
    crc = zlib.crc32(payload) & 0xFFFFFFFF
    header = struct.pack(HEADER_FMT, crc, len(key_bytes), len(value))
    offset = self._active_file.tell()
    self._active_file.write(header + payload)
    self._active_file.flush()  # line ~165 — same problem
    return offset
```

Search the entire file: zero calls to `os.fsync` or `os.fdatasync`. This means `put()` and `delete()` (which writes a tombstone) both return successfully with data only in the page cache.

## Contrast with the Correct Implementations

The codebase has three implementations that get this right:

**`write-ahead-log/wal.py`** — the `_do_sync` method at **lines 114–115**:
```python
self._fd.flush()
os.fsync(self._fd.fileno())
```
Every sync-mode write does both. Batch mode accumulates writes but still fsyncs before reporting the batch as durable (lines 127–128).

**`b-tree-storage-engine/btree.py`** — the WAL's `log_write` at **lines 136–137** and the PageManager's `sync` at **lines 104–105**:
```python
self._f.flush()
os.fsync(self._f.fileno())
```
The B-tree WAL fsyncs every entry before returning, and the PageManager syncs before the WAL is cleared — exactly the right ordering.

**`hash-index-storage/bitcask.py`** — `_write_record` at **lines 86–88**:
```python
self.active_file.flush()
if self.sync_writes:
    os.fsync(self.active_file.fileno())
```
This one gets it right *and* makes the tradeoff explicit with a `sync_writes` flag, mirroring how real Bitcask lets you choose between durability and throughput.

## Why This Matters for DDIA

Kleppmann's Chapter 3 makes the distinction between "the database reported success" and "the data is durable" central to storage engine design. These two bugs are the concrete implementation of that distinction — they silently break the durability contract while appearing to work perfectly under normal operation. You'll only discover the data loss after a crash, which is exactly when you need the data most.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:WAL.append` — Trace the full write path from LSMTree.put through the WAL to understand what's at risk during a crash
- [function] `b-tree-storage-engine/btree.py:WAL.commit` — Study how the B-tree WAL coordinates fsync ordering between the WAL and the data file (write-ahead protocol)
- [function] `write-ahead-log/wal.py:WriteAheadLog._do_sync` — Compare the three sync modes (sync, batch, none) to understand the durability-vs-throughput tradeoff spectrum
- [file] `log-structured-hash-table/bitcask.py` — Compare with `hash-index-storage/bitcask.py` side-by-side: same Bitcask design, one durable, one not
- [general] `directory-fsync` — Neither buggy nor correct implementations fsync the *parent directory* after creating new segment/WAL files — a subtler durability gap where the file's directory entry can be lost

## Beliefs

- `lsm-wal-missing-fsync` — The LSM tree WAL in `log-structured-merge-tree/lsm.py` calls `flush()` but never `os.fsync()`, making all WAL entries vulnerable to loss on power failure
- `log-hash-table-missing-fsync` — The log-structured hash table in `log-structured-hash-table/bitcask.py` calls `flush()` in `_write_record` but never `os.fsync()`, providing no crash durability for writes
- `flush-only-no-durability` — `file.flush()` moves data from Python buffers to the OS page cache but provides zero guarantee of persistence across power loss or kernel crash
- `correct-sync-pattern` — The WAL, B-tree, and hash-index-storage implementations all follow the same correct pattern: `flush()` immediately followed by `os.fsync(fd.fileno())` before returning to the caller
- `fdatasync-never-used` — No implementation in the codebase uses `os.fdatasync()`, missing a potential optimization for write-heavy paths where metadata updates aren't needed for recovery correctness

