# Topic: The gap between `_wal_files()` and `os.path.getsize()` is a TOCTOU window; explore whether this matters in practice given the single-writer design.

**Date:** 2026-05-29
**Time:** 12:14

## The TOCTOU Window in `_open_latest`

The race condition lives in `_open_latest` at `write-ahead-log/wal.py:101-109`:

```python
def _open_latest(self):
    files = self._wal_files()                          # line 102: snapshot directory listing
    if files:
        last = files[-1]
        if os.path.getsize(last) < self._max_file_size:  # line 105: stat the file
            self._current_file = last
            self._fd = open(last, "ab")                   # line 107: open for append
            return
    self._rotate()
```

Three filesystem operations happen sequentially with no atomicity guarantee:
1. **List** the WAL directory (`_wal_files()` at line 102 calls `os.listdir`)
2. **Stat** the last file (`os.path.getsize` at line 105)
3. **Open** it for append (line 107)

Between steps 1 and 2, the file `last` could be deleted — by a concurrent `truncate()` call that removes fully-consumed WAL segments (line 188 onwards). Between steps 2 and 3, the file could grow past `_max_file_size`, causing the writer to append to an already-oversized segment instead of rotating.

### Why it doesn't matter here

**`_open_latest` only runs during `__init__`** (line 79). At construction time, the WAL instance hasn't been handed to any caller yet, so no thread can be calling `append`, `truncate`, or `_rotate` concurrently against this same instance.

**The threading lock protects all mutation paths.** Every method that modifies WAL state — `append` (line 151), `append_batch` (line 161), `checkpoint` (line 177), `truncate` (line 185) — acquires `self._lock` (line 76). Since `_open_latest` runs before the constructor returns, the lock can't be contended.

**The single-writer design means one `WriteAheadLog` instance per directory.** There's no second instance that could be truncating files out from under the first. The test at `write-ahead-log/test_wal.py:33-44` (`test_crash_recovery`) creates a second instance on the same directory, but it does so sequentially — the first instance isn't actively writing when the second opens.

### Where it *would* matter

The design has no **inter-process** protection. If an external process (backup script, second application instance) deleted a WAL file between `_wal_files()` and `os.path.getsize()`, the `getsize` call would raise `FileNotFoundError` and crash the constructor. A production WAL would use advisory file locks (`fcntl.flock` or equivalent) on the directory to enforce single-writer semantics across processes, not just across threads.

Similarly, `_maybe_rotate` (line 147) uses `self._fd.tell()` instead of `os.path.getsize()`, which avoids the TOCTOU entirely for the hot path — the file position is in-process state, not a filesystem query. This is the right pattern; `_open_latest` uses `getsize` because it inherits a file it didn't create, so it doesn't have a valid `tell()` position yet.

### Bottom line

The TOCTOU window is real but inert. The single-writer-per-directory contract, combined with `_open_latest` running only at construction time, means no concurrent mutation can hit the gap. It's a textbook example of a race condition that's benign by design invariant rather than by synchronization — which is fine for a reference implementation, but would need hardening (directory-level flock) before use in production.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_maybe_rotate` — Uses `fd.tell()` instead of `getsize()`, eliminating TOCTOU on the hot write path; compare the two rotation-check strategies
- [function] `write-ahead-log/wal.py:truncate` — The truncation path also calls `_wal_files()` and then operates on those paths; trace whether the lock fully protects it
- [general] `inter-process-wal-safety` — How production WALs (SQLite, PostgreSQL) use advisory locks or PID files to enforce single-writer across processes, not just threads
- [function] `write-ahead-log/wal.py:_recover_seq_num` — Another init-time path that iterates `_wal_files()` and opens each one; same TOCTOU shape, same benign-by-construction argument
- [file] `hash-index-storage/bitcask.py` — Uses `os.path.getsize` at lines 122 and 208 in a segment-based storage engine; worth checking whether those have similar TOCTOU windows

---

## Beliefs

- `wal-open-latest-init-only` — `_open_latest` is called exclusively from `__init__`, so the TOCTOU window between `_wal_files()` and `os.path.getsize()` cannot be hit by concurrent WAL operations
- `wal-single-writer-thread-level` — The WAL enforces single-writer via `threading.Lock` but has no inter-process locking mechanism (no flock/PID file), so the single-writer invariant holds only within a single OS process
- `wal-hot-path-avoids-getsize` — `_maybe_rotate` uses `self._fd.tell()` instead of `os.path.getsize()`, avoiding filesystem stat calls and TOCTOU races on every write
- `wal-all-mutations-under-lock` — Every WAL method that writes records or modifies files (`append`, `append_batch`, `checkpoint`, `truncate`) acquires `self._lock` before performing any I/O

