# Topic: The three-way trade-off between metadata durability (`fsync`), data-only durability (`fdatasync`), and cache bypass (`O_DIRECT`) that DDIA Chapter 3 discusses

**Date:** 2026-05-29
**Time:** 11:05

# The `fsync` / `fdatasync` / `O_DIRECT` Trade-off

## What the codebase actually does

The short answer: **this codebase uniformly chooses `fsync` — the safest, slowest option — and never uses `fdatasync` or `O_DIRECT`.** That's a deliberate pedagogical choice, but understanding *why* production systems sometimes pick the other two is essential context.

Every storage engine in the repo follows the same pattern — `flush()` then `os.fsync()`:

- **WAL** (`write-ahead-log/wal.py:128-133`): The `_do_sync` method calls `self._fd.flush()` followed by `os.fsync(self._fd.fileno())`. This happens on every write in `"sync"` mode, or every N writes in `"batch"` mode.
- **B-tree PageManager** (`b-tree-storage-engine/btree.py:104-105`): `sync()` does `self._f.flush()` then `os.fsync(self._f.fileno())`. The B-tree's own WAL (`btree.py:136-137`) does the same on every `log_write`.
- **Bitcask** (`hash-index-storage/bitcask.py:88`): `_write_record` calls `os.fsync(self.active_file.fileno())` after every record when `sync_writes=True`.
- **LSM WAL** (`log-structured-merge-tree/lsm.py:26`): Notably, this one only calls `self._fd.flush()` — **no `fsync` at all**. This is a durability gap: `flush()` pushes data from Python's userspace buffer to the OS page cache, but the OS can still lose it on power failure.

## The three options explained

| Syscall | What it flushes | Metadata updated? | Relative cost |
|---------|----------------|-------------------|---------------|
| `fsync` | Data + metadata (size, mtime, directory entry) | Yes | Highest — two disk writes minimum |
| `fdatasync` | Data + *only* metadata needed for data retrieval (file size) | Partial — skips mtime, atime | ~50% cheaper on many filesystems |
| `O_DIRECT` | Bypasses page cache entirely; writes go straight to disk | No flush needed — never cached | Avoids double-buffering, but you manage alignment yourself |

### Why `fsync` is the safe default

`fsync` guarantees that after it returns, both the file's data *and* all its metadata are on stable storage. For a WAL, this is critical: if the file was just created or extended, the *directory entry* and *file size* must also be durable. Without metadata persistence, the OS might know the data blocks exist on disk but lose the directory link to them after a crash.

The WAL rotation code in `wal.py:114-116` illustrates why this matters — when rotating to a new file, the old file gets `flush()` + `fsync()` + `close()` before the new one opens. If that `fsync` were an `fdatasync`, and the old file had just been created, its directory entry might not be durable yet.

### Why production systems use `fdatasync`

For an append-only log that already exists (not newly created), the only metadata change on each write is `mtime` and `file size`. `fdatasync` will still flush the size change (since it's needed to find the data), but skips `mtime` — saving one metadata I/O. Databases like PostgreSQL use `fdatasync` as their default `wal_sync_method` for exactly this reason.

The WAL's `_do_sync` in `wal.py:128-133` would be a natural candidate for `fdatasync` — it's appending to an existing file whose directory entry is already stable. The Bitcask engine at `bitcask.py:88` is another: it only appends to an open active file.

### Why some systems use `O_DIRECT`

`O_DIRECT` eliminates the page cache entirely. The application writes directly to disk, avoiding the double-copy problem (userspace buffer → page cache → disk). This is valuable when:

1. The application has its own buffer pool (like a B-tree page cache — see `PageManager` in `btree.py:27-113`, which manages its own pages)
2. You want predictable latency (no interference from the OS flushing dirty pages at inconvenient times)
3. You're doing large sequential I/O that won't benefit from caching

The B-tree's `PageManager` is the strongest candidate for `O_DIRECT` in this codebase — it already manages fixed-size pages and knows which ones are dirty. But `O_DIRECT` comes with painful constraints: writes must be sector-aligned (typically 512B or 4KB), and the buffers must be aligned in memory. Python's file I/O doesn't expose this easily, which is likely why the implementations avoid it.

## The durability gap in the LSM WAL

The LSM tree's WAL (`lsm.py:26`) only calls `self._fd.flush()` without a subsequent `os.fsync()`. This means data reaches the OS page cache but is **not guaranteed to be on disk**. A power failure after `flush()` but before the OS writes back dirty pages would lose the WAL entry. Compare this to the standalone WAL (`wal.py:128-133`) which always calls `os.fsync()`. This is either a bug or an intentional simplification — either way, it's a real durability difference.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_do_sync` — The sync-mode dispatch logic; compare "sync" vs "batch" trade-offs and where `fdatasync` would fit
- [function] `b-tree-storage-engine/btree.py:PageManager.sync` — B-tree page flush path; consider how `O_DIRECT` would change this architecture
- [general] `directory-fsync-gap` — None of these engines `fsync` the parent *directory* after creating a new file — a subtle durability gap that ext4 exposes
- [function] `log-structured-merge-tree/lsm.py:WAL.append` — The LSM WAL's missing `fsync` — trace what happens to durability guarantees during crash recovery
- [general] `fdatasync-vs-fsync-benchmarking` — Measure the actual latency difference on your filesystem; on ext4 with `data=ordered`, `fdatasync` can be 2x faster for append workloads

## Beliefs

- `fsync-only-durability` — All three storage engines (WAL, B-tree, Bitcask) use exclusively `os.fsync` for durability; neither `fdatasync` nor `O_DIRECT` appears anywhere in the codebase
- `lsm-wal-missing-fsync` — The LSM tree's WAL (`lsm.py:26`) calls only `flush()` without `os.fsync()`, making it vulnerable to data loss on power failure unlike the standalone WAL
- `flush-before-fsync-invariant` — Every `os.fsync()` call in the codebase is preceded by a `flush()` call, ensuring Python's userspace buffers are drained to the OS before requesting stable storage persistence
- `bitcask-sync-writes-flag` — Bitcask's `fsync` behavior is controlled by a `sync_writes` constructor parameter (`bitcask.py:30`), making durability configurable per-instance — the only engine that exposes this as a knob
- `wal-batch-sync-deferred-durability` — The standalone WAL supports a `"batch"` sync mode (`wal.py:130-133`) that defers `fsync` until N writes accumulate, trading durability of individual records for throughput

