# Topic: SQLite's approach to directory fsyncs and the `SQLITE_FCNTL_SYNC` mechanism as a reference implementation of correct behavior

**Date:** 2026-05-29
**Time:** 11:23

I have enough material. Here's the explanation:

---

# SQLite's Directory Fsync and `SQLITE_FCNTL_SYNC` as a Reference for Correct Durability

## Why This Matters

The DDIA reference implementations contain a systematic gap: **none of the 13 `os.fsync()` calls across the entire codebase target a directory file descriptor**. Every fsync operates on a regular file — WAL segments, B-tree page files, Bitcask data files. The *directory entries* that map filenames to inodes are never explicitly fsynced. This is precisely the problem that production databases like SQLite solve, and understanding how SQLite does it illuminates what's missing here.

## The POSIX Split: File Data vs. Directory Metadata

When you create a new file or rename an existing one, two independent metadata structures change:

1. **The file's inode** — data blocks, size, timestamps. Flushed by `fsync(file_fd)`.
2. **The parent directory's entry** — the name→inode mapping. Flushed by `fsync(dir_fd)`.

These are separate, and POSIX makes no promise that fsyncing the file also flushes the directory. This is documented extensively in `entries/2026/05/29/topic-posix-durability-guarantees.md` (lines 26–42), which confirms that all 13 fsync sites in the codebase target only file descriptors.

The practical consequence: a newly created file can have its *contents* safely on disk while its *directory entry* is still in the page cache. A crash loses the filename. The inode becomes an orphan. Recovery code that uses `os.listdir()` — as every recovery method in this codebase does (`entries/2026/05/29/topic-directory-fsync.md`, line 62) — simply never sees the file. Silent data loss, no corruption signal.

## How SQLite Handles This

SQLite treats directory fsync as a first-class operation in its VFS (Virtual File System) layer. The key mechanisms:

### 1. The `unixSync` / `openDirectory` Pattern

SQLite's Unix VFS (in `os_unix.c`) opens the parent directory after creating or renaming a file and calls `fsync()` on that directory file descriptor. This is the `_fsync_directory` helper that `entries/2026/05/29/topic-directory-fsync.md` (lines 64–75) describes as the missing fix:

```python
def _fsync_directory(dir_path):
    fd = os.open(dir_path, os.O_RDONLY)
    try:
        os.fsync(fd)
    finally:
        os.close(fd)
```

SQLite calls this pattern after every operation that changes directory metadata — journal file creation, WAL file creation, database file renames during commit.

### 2. `SQLITE_FCNTL_SYNC`

The `SQLITE_FCNTL_SYNC` file control is SQLite's mechanism for signaling the VFS layer that a sync operation is about to happen or has completed. It's part of SQLite's layered approach to separating sync *policy* (when to sync) from sync *mechanism* (how to sync on this platform). The VFS implementation decides whether `fsync`, `fdatasync`, or `F_FULLFSYNC` (on macOS) is appropriate — and crucially, whether the directory also needs syncing.

This is the abstraction that the DDIA implementations lack entirely. As noted in `entries/2026/05/29/topic-posix-durability-guarantees.md` (lines 76, 116), the codebase runs on macOS where `os.fsync()` doesn't even guarantee flushing the disk write cache — only `fcntl(fd, F_FULLFSYNC)` does that. SQLite's VFS handles this transparently through its `SQLITE_FCNTL_SYNC` / `F_FULLFSYNC` integration.

### 3. The Write-Temp / Fsync / Rename / Fsync-Dir Protocol

SQLite, PostgreSQL, and RocksDB all follow a four-step protocol for safely replacing files, as described in `entries/2026/05/29/topic-posix-durability-guarantees.md` (line 108):

1. Write data to a temporary file
2. `fsync` the temporary file (content is durable)
3. `rename` the temp file to the final name (atomic but not durable)
4. `fsync` the parent directory (rename is now durable)

No implementation in this codebase follows this protocol. Both Bitcask variants call `os.rename()` during compaction — `hash-index-storage/bitcask.py:297` and `log-structured-hash-table/bitcask.py:301` — without the final directory fsync (`entries/2026/05/29/topic-directory-fsync-after-rename.md`, lines 25–28).

## Where the Gap Is Most Dangerous

The observations identify a compound failure mode that is the most critical finding. From `entries/2026/05/29/topic-shadow-paging-or-steal-no-force.md` (line 121):

> The most dangerous compound failure: SSTable created (without directory fsync) → WAL truncated → crash → both gone.

In the LSM tree (`lsm.py`), flushing the memtable creates a new SSTable file. Then the WAL is truncated. If the SSTable's directory entry was never fsynced, a crash after truncation loses *both* — the WAL records are gone (truncated) and the SSTable is invisible (directory entry lost). The data existed in exactly one place (the page cache) and vanished.

SQLite prevents this by ensuring every new file's directory entry is fsynced before the journal/WAL is invalidated. The ordering is explicit: new file durable → journal cleared. This is the cross-file ordering protocol that `entries/2026/05/29/topic-fsync-ordering-guarantees.md` (line 85) identifies as absent from the entire codebase.

## Why Developers Miss This

Three factors conspire to make this bug invisible (`entries/2026/05/29/topic-directory-fsync.md`, lines 58–63):

1. **File data *is* fsynced.** The WAL, B-tree, and Bitcask implementations are thorough about fsyncing file contents. Seeing the `os.fsync()` call creates a false sense of security.
2. **ext4's `data=ordered` mode masks it.** The default Linux filesystem flushes directory metadata within ~5 seconds. The vulnerability window is real but narrow. On XFS or ext4 with `data=writeback`, the window is much wider.
3. **macOS APFS masks it further.** APFS's copy-on-write transaction model provides implicit rename durability, so the bug never manifests on the development platform (Darwin 24.4.0). It's a latent Linux-only defect.

SQLite is fastidious about this precisely because it's been burned by it across every filesystem and platform over two decades.

## What SQLite Teaches for This Codebase

The core lesson is that SQLite treats "making data durable" as a multi-layer problem:

| Layer | SQLite | This Codebase |
|-------|--------|---------------|
| File content → disk | `fsync` / `fdatasync` / `F_FULLFSYNC` via VFS | `os.fsync()` (correct on Linux, incomplete on macOS) |
| Directory entry → disk | `openDirectory` + `fsync(dir_fd)` | **Missing entirely** |
| Cross-file ordering | Explicit fsync sequence: WAL → data → directory | No protocol; caller-dependent |
| Platform abstraction | VFS layer with `SQLITE_FCNTL_SYNC` | Hardcoded `os.fsync()` everywhere |

The fix is not large — a 5-line helper function called at each file-creation and rename site — but it requires recognizing that durability is not just about file content.

---

## Topics to Explore

- [general] `sqlite-vfs-layer` — SQLite's Virtual File System abstraction (`os_unix.c`) that encapsulates platform-specific sync behavior including `F_FULLFSYNC`, `fdatasync`, and directory fsync behind a single API
- [function] `log-structured-merge-tree/lsm.py:_flush` — The most dangerous path in the codebase: SSTable creation with no fsync on file or directory, followed by WAL truncation, creating a window where both can be lost
- [general] `ext4-auto-da-alloc` — How ext4's `auto_da_alloc` feature (since kernel 2.6.30) partially mitigates the rename-without-directory-fsync problem for replace-via-rename, but does *not* help for new file creation — understanding when the filesystem saves you and when it doesn't
- [function] `write-ahead-log/wal.py:_rotate` — WAL rotation creates new files at line 119 whose directory entries are never fsynced; compare with SQLite's journal file creation protocol
- [general] `macos-f-fullfsync-semantics` — How `fcntl(fd, F_FULLFSYNC)` differs from `fsync` on Darwin/APFS, and why SQLite's VFS uses it while this codebase's 13 fsync calls may provide no power-loss durability on the current platform

## Beliefs

- `sqlite-fsyncs-directories-after-file-ops` — SQLite's Unix VFS calls `fsync` on the parent directory file descriptor after every file creation, rename, and journal/WAL lifecycle event, ensuring directory metadata reaches stable storage before dependent operations proceed
- `no-directory-fsync-anywhere` — All 13 `os.fsync()` calls across the DDIA implementations target regular file descriptors; no code anywhere opens a directory file descriptor or calls fsync on one, leaving every file creation and rename non-durable at the directory level
- `sqlite-fcntl-sync-abstracts-platform-differences` — `SQLITE_FCNTL_SYNC` is part of SQLite's VFS abstraction that separates sync policy from mechanism, allowing the same durability logic to use `fsync`, `fdatasync`, or `F_FULLFSYNC` depending on the platform — a layer entirely absent from this codebase
- `lsm-flush-then-truncate-is-compound-failure` — The LSM tree's flush path creates an SSTable (no directory fsync) then truncates the WAL, creating a window where a crash loses both the WAL records and the SSTable's directory entry, resulting in silent data loss with no recovery signal
- `atomic-rename-is-not-durable-rename` — POSIX guarantees that `os.rename()` is atomic (all-or-nothing at crash) but not durable; without fsyncing the parent directory after rename, a crash can silently revert the rename on ext4/XFS, which both Bitcask compaction paths rely on without the directory fsync

