# Topic: The lack of atomic rename means a crash mid-truncate can lose records; compare with how real databases (e.g., SQLite, PostgreSQL) handle WAL truncation using temp files or header flags

**Date:** 2026-05-29
**Time:** 06:23

No problem — I have enough from the observations to give a thorough explanation.

---

# Crash-Unsafe WAL Truncation: The Missing Atomic Rename

## The Vulnerability

Look at `truncate()` in `write-ahead-log/wal.py:179`. The method does this:

1. Flushes and closes the current file descriptor
2. Iterates over every `.wal` file, reading all records
3. Filters out records with `seq_num <= up_to_seq`
4. **Rewrites each file in place** with only the kept records (or deletes the file if nothing is kept)

The critical code path from the observation starts at line 179:

```python
def truncate(self, up_to_seq: int) -> None:
    """Remove all records with seq_num <= up_to_seq."""
    with self._lock:
        if self._fd:
            self._fd.flush()
            os.fsync(self._fd.fileno())
            self._fd.close()
            self._fd = None

        for path in self._wal_files():
            kept = []
            with open(path, "rb") as f:
                while True:
                    try:
                        rec = _read_record(f)
                        if rec is None:
                            break
                        if rec.seq_num > up_to_seq:
                            kept.append(rec)
                    except ValueError:
                        break
            if not kept:
                # deletes the file entirely
```

The rest of the method (not shown but inferrable) rewrites the file with only the `kept` records. Here's why that's dangerous:

### The Crash Window

If the process crashes **after** the old file content has been truncated/overwritten but **before** the new content is fully written and fsynced, you lose records from both sets:

- Records with `seq_num <= up_to_seq` are already gone (the old file was opened for rewrite)
- Records with `seq_num > up_to_seq` haven't finished writing yet

This is a **total data loss window** for that WAL file. The CRC checksums (`_read_record` at line 43-54) will catch the corruption on recovery — the partial write will fail CRC validation — but catching corruption and preventing data loss are very different things. You'll know records are missing, but you can't get them back.

### The Same Pattern in the LSM Tree

The simpler WAL in `log-structured-merge-tree/lsm.py:56` has an even more aggressive version:

```python
def truncate(self):
    self._fd.close()
    self._fd = open(self._path, "wb")   # instantly destroys all content
    self._fd.close()
    self._fd = open(self._path, "ab")
```

This opens the file with `"wb"` — write mode — which truncates the file to zero bytes immediately. A crash between the `open(path, "wb")` and the subsequent close/reopen leaves an empty WAL. If the memtable flush to SSTable wasn't fully durable yet, those records are gone.

## How Real Databases Solve This

### SQLite: Header Flags (Rollback Journal Mode)

SQLite never modifies the WAL in a way that destroys data before the replacement is confirmed durable. In WAL mode, SQLite uses a **WAL index** (the `-shm` file) and header checksums to track how far the WAL has been applied. Truncation works by:

1. Checkpointing all WAL frames into the main database file
2. Fsyncing the database file
3. Only *then* resetting the WAL — either by rewriting just the WAL header (with a salt change that invalidates old frames) or truncating the file

The key insight: the WAL header contains a **salt value**. Changing the salt atomically invalidates all existing frames without needing to physically remove them. A crash mid-checkpoint simply means the WAL frames are replayed again on next open — idempotent replay, no data loss.

### PostgreSQL: Segment Recycling

PostgreSQL's WAL uses fixed-size 16 MB segment files (e.g., `000000010000000000000001`). When segments are no longer needed:

1. The checkpoint process writes all dirty buffers to data files
2. Fsyncs the data files
3. Updates `pg_control` (a small file with a known layout) to record the new checkpoint location
4. Old WAL segments are **renamed** (recycled) rather than rewritten — `os.rename()` is atomic on POSIX systems

The rename-based approach means a segment file is either fully the old content or fully the new content. There's no window where it's half-written. PostgreSQL also keeps `pg_control` as the single source of truth for "what has been checkpointed" — if the control file update doesn't complete, recovery simply replays more WAL than strictly necessary.

### The Write-Rename-Fsync Pattern

The standard safe pattern that both systems use in various forms:

```python
def safe_truncate(self, up_to_seq):
    for path in self._wal_files():
        kept = self._filter_records(path, up_to_seq)
        if not kept:
            os.unlink(path)  # atomic on POSIX
        else:
            tmp = path + ".tmp"
            self._write_records(tmp, kept)
            os.fsync(...)           # ensure tmp is durable
            os.rename(tmp, path)    # atomic replacement
            os.fsync(dir_fd)        # ensure directory entry is durable
```

`os.rename()` on POSIX is atomic — the file is either the old version or the new version, never a partial mix. This is the fundamental primitive that this implementation is missing.

### What the Implementation Could Add

For this teaching codebase, the fix would be straightforward — the `truncate()` method should:

1. Write kept records to a temporary file (e.g., `000001.wal.tmp`)
2. `fsync` the temp file
3. `os.rename(tmp, original)` — atomic swap
4. `fsync` the directory file descriptor to ensure the rename is durable

Alternatively, following SQLite's approach: don't rewrite WAL files at all. Instead, maintain a durable "truncation watermark" (like a checkpoint record, which this WAL already supports at line 169) and simply skip records below the watermark during replay. Old files can then be unlinked wholesale once all their records fall below the watermark.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:checkpoint` — The checkpoint record (line 169) could serve as a durable truncation watermark, eliminating the need for in-place rewrites entirely
- [function] `log-structured-merge-tree/lsm.py:truncate` — Compare the LSM WAL's even more aggressive truncation (line 56) which opens with `"wb"` and instantly destroys all content
- [file] `hash-index-storage/bitcask.py` — The Bitcask compaction at line 297 uses `os.rename()` for segment rotation — contrast this safe approach with the WAL truncation
- [general] `fsync-directory-durability` — Even with atomic rename, the directory entry must be fsynced separately on Linux for the rename to survive a power loss; none of the implementations in this repo do this
- [function] `write-ahead-log/wal.py:_read_record` — The CRC validation (lines 43-54) detects post-crash corruption but doesn't prevent data loss — understanding the difference between detection and prevention is key

## Beliefs

- `wal-truncate-not-crash-safe` — `WriteAheadLog.truncate()` rewrites WAL files in place without write-to-temp-then-rename, creating a window where a crash loses both old and new records
- `lsm-wal-truncate-destroys-immediately` — `LSMTree`'s `WAL.truncate()` opens the file with `"wb"` which zeroes it instantly, with no intermediate durable state to recover from
- `crc-detects-but-does-not-prevent` — CRC32 checksums in `_read_record` detect partial writes after a crash but cannot recover the lost data; they turn silent corruption into detected data loss
- `checkpoint-record-exists-but-unused-for-truncation` — The WAL supports `OP_CHECKPOINT` records (line 169) that could serve as a durable truncation marker, but `truncate()` physically removes records instead of using this mechanism
- `no-directory-fsync-anywhere` — No implementation in the repo calls `fsync` on a directory file descriptor after file creation or rename, meaning even correct atomic-rename patterns would not be fully crash-safe on Linux

