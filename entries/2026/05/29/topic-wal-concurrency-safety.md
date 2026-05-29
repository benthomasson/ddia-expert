# Topic: The lock is released before iteration begins; explore what guarantees (or lack thereof) exist for concurrent read/write access

**Date:** 2026-05-29
**Time:** 06:32

Now I have a thorough understanding of the concurrency picture. Here's the explanation:

---

# Lock Released Before Iteration: Concurrent Read/Write Guarantees

## The Pattern

Both `iterate()` and `replay()` in `write-ahead-log/wal.py` follow the same concurrency pattern: **acquire lock → flush → release lock → read**. The lock protects only the flush call, not the subsequent file reads.

From the `iterate()` entry (`entries/2026/05/28/write-ahead-log-wal-iterate.md`, lines 37–38):

> Grabs `_lock`, checks if there's an open file descriptor (`_fd`), and flushes it. [...] The lock is released immediately after the flush — it is **not** held during iteration.

And from `replay()` (`entries/2026/05/28/write-ahead-log-wal-replay.md`, line 38):

> Acquires `self._lock` and flushes the file descriptor. [...] The lock is released immediately — the read phase is lock-free.

The underlying `_read_all_records()` method does not acquire `self._lock` at all (`entries/2026/05/29/write-ahead-log-wal-_read_all_records.md`, line 104): it trusts the caller to have flushed first.

## What the Flush Guarantees

The flush under lock provides one guarantee: **all `append()` calls that returned before you called `iterate()`/`replay()` will have their bytes visible in the OS page cache**. This means you won't miss writes that your own thread (or another thread) completed before the read started.

But note that `iterate()` calls `fd.flush()`, not `os.fsync()` (`entries/2026/05/28/write-ahead-log-wal-iterate.md`, line 43). The data reaches the kernel page cache but isn't guaranteed durable on disk. For iteration purposes this doesn't matter — we're reading via the same kernel — but it's a subtlety worth knowing.

## What's NOT Guaranteed

### 1. No snapshot isolation

Once the lock is released and `_read_all_records()` begins iterating files, concurrent `append()` calls can write new records to the WAL. The iterator may or may not see them:

- If the new record lands in a file the iterator **hasn't opened yet**, it will likely be visible.
- If the new record lands in the file the iterator **is currently reading**, the behavior depends on OS-level buffering and file position.
- If the new record lands in a file the iterator **has already finished reading**, it won't be seen.

There is no deterministic contract here. The `iterate-not-snapshot-isolated` belief captures this directly: *"concurrent appends may or may not be visible to the iterator."*

### 2. Partial reads of in-flight writes

`replay()` has an additional hazard (`entries/2026/05/28/write-ahead-log-wal-replay.md`, line 103): *"concurrent `append` calls during replay could produce a partial read of in-flight writes."*

This can happen because `append()` and `append_batch()` hold the lock during their `write()` call, but the iterator reads without the lock. If the iterator reads a file at the exact moment a write is in progress, it could see:
- A partially written length prefix → `_read_record` returns `None` (treated as EOF)
- A complete length prefix but partial payload → `_read_record` returns `None`
- A complete record with bad CRC → `_read_record` raises `ValueError`, and `_read_all_records` **terminates the entire iterator**

The last case is the dangerous one: a concurrent write could cause corruption-stop behavior, silently dropping all remaining valid records across all subsequent files.

### 3. No protection against concurrent structural mutations

The WAL's `truncate()` method rewrites files on disk. If `truncate()` runs concurrently with `iterate()` or `replay()`, the iterator could:
- Try to open a file that has been deleted
- Read a file that is being rewritten mid-read
- See a mix of old and new content

Similarly, `_maybe_rotate()` (called from `append`/`append_batch` after writes) creates new files and closes the current one. An iterator scanning `_wal_files()` at the start might not see the new file, or might see a half-written file if rotation happens mid-iteration.

## The LSM Tree: Even Fewer Guarantees

The LSM tree in `log-structured-merge-tree/lsm.py` provides a useful contrast. It has **no locking at all** — no `threading.Lock`, no atomic swaps, no synchronization primitives (`entries/2026/05/28/topic-lsm-concurrency-safety.md`, line 76).

Its `range_scan()` at line 272 iterates `self._memtable` and `self._sstables` without any protection. Three concrete failure modes exist:

| Concurrent operation | What breaks |
|---------------------|-------------|
| `_flush()` during `range_scan()` | Memtable cleared mid-iteration → `RuntimeError` or duplicate results (data appears in both memtable and new SSTable) |
| `compact()` during `range_scan()` | Old SSTable files deleted → `FileNotFoundError` or stale data (deleted keys resurface) |
| `_flush()` during `compact()` | New SSTable may be included in or lost from the compaction merge → data duplication or data loss |

The test suite (`test_lsm.py`) is entirely single-threaded, so these races are never exercised.

## Why This Design Is Acceptable

This is a **reference implementation** for DDIA concepts, not production code. The WAL's lock-release-before-iteration design reflects a deliberate trade-off:

- **Holding the lock during iteration would block all writers** for the entire duration of the read scan, which could be long (every `.wal` file on disk). This would make the WAL nearly unusable for concurrent access.
- **The flush-under-lock idiom** provides the minimum useful guarantee: "you'll see everything that was committed before you started reading." This matches the typical crash-recovery use case, where `replay()` is called during startup before concurrent writes begin.
- **Production WALs** (SQLite, RocksDB, etc.) solve this with more sophisticated mechanisms: immutable segments, copy-on-write data structures, reference counting on file handles, and MVCC. These are the techniques described in DDIA Chapter 3 but intentionally simplified in this codebase.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_all_records` — The generator that does the actual file-by-file scan; its `break` vs `return` distinction on errors determines whether concurrent-write damage is localized or catastrophic
- [function] `write-ahead-log/wal.py:truncate` — Rewrites WAL files in place without atomic rename; concurrent iteration during truncation is a compounding hazard on top of the lock-release pattern
- [function] `log-structured-merge-tree/lsm.py:_flush` — The memtable-to-SSTable promotion that creates the most dangerous concurrency window: data briefly exists in neither location
- [general] `mvcc-snapshot-isolation` — DDIA's discussion of how production systems solve this: readers get an immutable view while writers create new versions, eliminating the lock-or-race dilemma entirely
- [general] `wal-atomicity-guarantees` — Whether a single `write()` call actually provides atomicity on common filesystems (ext4, APFS) — the assumption underlying `append_batch`'s safety

## Beliefs

- `wal-iterate-replay-lock-scope` — Both `iterate()` and `replay()` hold `self._lock` only during `fd.flush()`, not during the subsequent file reads via `_read_all_records()`; the read phase is entirely lock-free
- `wal-concurrent-write-during-read-can-trigger-corruption-stop` — A concurrent `append()` writing to a file the iterator is reading can produce a partial record that `_read_record` interprets as CRC failure, terminating the entire iterator and dropping all remaining valid records
- `lsm-range-scan-zero-concurrency-protection` — `range_scan()` in `lsm.py` has no locks, no snapshots, and no atomic swaps; concurrent `_flush()` or `compact()` can cause duplicate results, missing data, or crashes
- `wal-flush-not-fsync-before-iteration` — `iterate()` and `replay()` call `fd.flush()` but not `os.fsync()` before reading, so buffered writes reach the page cache (sufficient for same-process reads) but are not guaranteed durable

