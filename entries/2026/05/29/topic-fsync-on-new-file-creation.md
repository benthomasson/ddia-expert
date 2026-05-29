# Topic: Creating a new file (e.g., new WAL segment in `_rotate()`) also modifies the parent directory; none of the `_rotate` paths fsync the directory after creating the new segment

**Date:** 2026-05-29
**Time:** 08:32

## The Missing Directory `fsync` in `_rotate()`

When a filesystem creates a new file, two things happen at the storage level: (1) the file's data and metadata are written, and (2) the **parent directory's entry table** is updated to include the new filename. These are separate operations, and `fsync`ing the new file only guarantees the file's own data is durable — it does **not** guarantee the directory entry pointing to that file is durable.

### Where the bug lives

In `write-ahead-log/wal.py:111-120`, `_rotate()` does the following:

1. **Flushes and fsyncs the old segment** (lines 113–115) — this is correct
2. **Creates a new `.wal` file** via `open(..., "ab")` (line 120) — this modifies the parent directory
3. **Never fsyncs the directory** — the new file's directory entry can be lost

```python
def _rotate(self):
    """Create a new WAL file."""
    if self._fd:
        self._fd.flush()
        os.fsync(self._fd.fileno())   # fsyncs OLD file — good
        self._fd.close()
    # ... compute next filename ...
    self._current_file = os.path.join(self._dir, f"{next_num:06d}.wal")
    self._fd = open(self._current_file, "ab")  # creates file, NO dir fsync
```

### Why this matters for crash recovery

Consider this sequence:

1. WAL segment `000001.wal` fills up, `_rotate()` is called
2. The old segment is fsynced and closed — durable on disk
3. New segment `000002.wal` is created and opened
4. Records are written to `000002.wal` and fsynced via `_do_sync()`
5. **Power loss**

Without a directory fsync, the filesystem may have durably written the *contents* of `000002.wal` (because `os.fsync` was called on its fd), but the directory entry linking the name `000002.wal` to that inode may not have reached disk. On recovery, `_wal_files()` (line 82) calls `os.listdir(self._dir)` — and `000002.wal` simply won't appear. Those records are gone.

### All call sites are affected

Every path into `_rotate()` has this gap:

| Caller | Location | Context |
|--------|----------|---------|
| `_open_latest()` | line 109 | Startup when no file exists or all are full |
| `_maybe_rotate()` | line 139 | After `append()` (line 150), `append_batch()` (line 166), `checkpoint()` (line 176) |

None of these callers add a directory fsync either. The fix would be:

```python
def _fsync_directory(self):
    fd = os.open(self._dir, os.O_RDONLY)
    try:
        os.fsync(fd)
    finally:
        os.close(fd)
```

…called in `_rotate()` immediately after opening the new segment file.

### The same class of bug in other implementations

The grep results show `hash-index-storage/bitcask.py:101` and `log-structured-hash-table/bitcask.py:129` also have rotation methods. Across the entire codebase, there are zero calls to `os.fsync` on a directory fd — the pattern `os.open(dir, os.O_RDONLY)` followed by `os.fsync` appears nowhere. This suggests the directory fsync gap is systematic, not isolated to the WAL.

### Context from DDIA

This is exactly the class of durability subtlety Kleppmann discusses in Chapter 3. The POSIX `fsync` contract is per-fd: it guarantees the data reachable through *that file descriptor* is durable, but directory entries are metadata of the *directory*, not the file. Production systems like PostgreSQL, SQLite, and LevelDB all fsync the parent directory after creating new files.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:truncate` — Also creates/removes files during WAL cleanup; check whether it fsyncs the directory after unlinking segments
- [file] `hash-index-storage/bitcask.py` — Has its own `_maybe_rotate` at line 101; likely has the same missing directory fsync
- [file] `log-structured-hash-table/bitcask.py` — `_rotate_segment` at line 129; third instance of the same pattern to audit
- [function] `log-structured-merge-tree/lsm.py:SSTable.write` — Creates new SSTable files; check whether the directory is fsynced after writing a new table
- [general] `fsync-durability-guarantees` — Broader audit of all `os.fsync` calls across the repo to verify they cover both file data and directory entries

---

## Beliefs

- `wal-rotate-no-dir-fsync` — `_rotate()` in `write-ahead-log/wal.py` creates a new segment file but never calls `os.fsync` on the parent directory fd, so the new filename may not survive a crash
- `no-dir-fsync-anywhere` — No file in the repository opens a directory fd for fsync; the `os.open(dir, O_RDONLY)` + `os.fsync` pattern is entirely absent
- `wal-recovery-depends-on-listdir` — `_recover_seq_num()` and `_open_latest()` discover WAL segments via `os.listdir`, so any segment whose directory entry wasn't fsynced will be invisible after a crash
- `rotate-called-from-three-paths` — `_rotate()` is reachable from `_open_latest` (startup), `append`, `append_batch`, and `checkpoint` via `_maybe_rotate`, meaning all write paths are exposed to this durability gap

