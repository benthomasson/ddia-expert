# Topic: The current truncate rewrites files in place without atomic rename; explore what a crash-safe truncation would look like (write-to-temp + fsync + rename)

**Date:** 2026-05-28
**Time:** 18:20

I now have a thorough understanding of the codebase. Let me write the explanation.

---

# Crash-Safe WAL Truncation: From In-Place Rewrite to Atomic Rename

## The Current Implementation

The WAL's `truncate(up_to_seq)` method in `write-ahead-log/wal.py` follows this sequence (documented at lines 106–109 of the entry):

1. Flush, fsync, and **close** the current active file
2. Scan each `.wal` file and collect records with `seq_num > up_to_seq`
3. **Delete** files that become empty; **rewrite in place** files that have remaining records
4. Call `_open_latest()` to resume appending

The critical danger is step 3. When a WAL file contains a mix of truncatable and surviving records, the implementation reads the survivors, then writes them back to the *same file path*. This is a destructive, non-atomic operation.

## What Goes Wrong on Crash

Consider a WAL file `000003.wal` containing records with seq 15–25, and a `truncate(20)` call that should keep seq 21–25:

| Crash point | State on disk | Consequence |
|---|---|---|
| After `open(f, "wb")` but before any write | `000003.wal` is **empty** (truncated to zero by the `"wb"` mode) | Records 21–25 are **permanently lost**. The WAL is now shorter than reality. |
| After partial write of survivors | `000003.wal` has records 21–23 but not 24–25 | Two committed records silently vanish. Recovery replays a truncated log. |
| After full write but before fsync | Data is in the OS page cache, not on disk | A power failure drops the rewrite entirely — same as the empty-file case. |

In all three cases, the WAL loses data that should have survived. Since the WAL is the *sole durability guarantee* for the storage engines that depend on it (LSM memtable, B-tree page writes), this means acknowledged mutations can be silently lost.

The entry explicitly calls this out as an invariant: **"Truncation rewrites files in place. This is not crash-safe"** (entry line 117).

## The Fix: Write-to-Temp + Fsync + Rename

The standard technique from production systems (LevelDB, RocksDB, PostgreSQL, etcd) is a three-step atomic replacement:

```python
def truncate(self, up_to_seq: int) -> None:
    with self._lock:
        self._fd.flush()
        os.fsync(self._fd.fileno())
        self._fd.close()

        for wal_file in sorted(glob.glob(os.path.join(self._log_dir, "*.wal"))):
            records = list(self._read_records_from(wal_file))
            survivors = [r for r in records if r.seq_num > up_to_seq]

            if not survivors:
                os.unlink(wal_file)
                continue

            if len(survivors) == len(records):
                continue  # nothing to truncate in this file

            # Step 1: Write survivors to a temp file in the SAME directory
            tmp_path = wal_file + ".tmp"
            tmp_fd = os.open(tmp_path, os.O_WRONLY | os.O_CREAT | os.O_TRUNC)
            try:
                for rec in survivors:
                    os.write(tmp_fd, _encode_record(
                        rec.seq_num, OP_BYTES[rec.op_type],
                        rec.key, rec.value
                    ))
                # Step 2: Fsync the temp file — data is durable
                os.fsync(tmp_fd)
            finally:
                os.close(tmp_fd)

            # Step 3: Atomic rename replaces old file
            os.rename(tmp_path, wal_file)

        # Fsync the directory to make the rename durable
        dir_fd = os.open(self._log_dir, os.O_RDONLY)
        try:
            os.fsync(dir_fd)
        finally:
            os.close(dir_fd)

        self._open_latest()
```

### Why Each Step Matters

**Same-directory temp file.** `os.rename()` is only guaranteed atomic when source and destination are on the same filesystem. Writing the temp file alongside the WAL files ensures this.

**Fsync before rename.** Without the fsync on the temp file, the rename could succeed (updating the directory entry) while the temp file's data is still in the page cache. A power failure would leave a valid filename pointing to an empty or partial file — the same problem we started with.

**Fsync the directory after rename.** On Linux/ext4 and macOS/APFS, `rename()` updates a directory entry. That directory metadata change also needs to be flushed. Without this, a crash could revert the rename, leaving the old file in place — which is actually safe (you'd just re-truncate on next startup), but the directory fsync guarantees exactly-once truncation.

### Crash Safety Analysis

| Crash point | State on disk | Recovery |
|---|---|---|
| During temp file write | Original `.wal` intact, `.tmp` partial | Delete `.tmp` on startup, original data preserved |
| After temp fsync, before rename | Both `.wal` (old) and `.tmp` (new) exist | Either rename completes on startup, or delete `.tmp` and re-truncate |
| After rename, before dir fsync | Rename may or may not be visible | Either the new file is in place (done) or the old file is (re-truncate) |

In every scenario, the original data is never destroyed until the replacement is fully durable. This is the key invariant: **the old file and the new file coexist until the rename atomically swaps them**.

## How the B-Tree Engine Gets This Right (and Wrong)

The B-tree's embedded WAL in `b-tree-storage-engine/btree.py` takes a different approach. Its `commit()` method (documented at entry line 34) does:

1. Fsync the **data file** (ensuring all replayed page writes are durable)
2. **Truncate** the WAL (since all entries have been applied)

This is safe because the B-tree WAL truncation happens only *after* the data file is authoritative. If a crash happens mid-truncation, recovery simply replays the WAL entries again — they're idempotent page writes. The B-tree WAL never needs to *partially* truncate; it's all-or-nothing.

The standalone WAL doesn't have this luxury. It supports partial truncation (keep records above a threshold), so it must handle the mixed-file case where some records survive and others don't.

## The Bitcask Parallel

The Bitcask compaction in `hash-index-storage/bitcask.py` has a similar crash window (entry lines 91–97). During compaction, it:

1. Writes merged data to new files
2. Deletes old immutable files
3. Renumbers the active file

A crash between steps 2 and 3 could lose the active file's identity. The same write-to-temp + rename pattern would fix this — write the merged files atomically, then delete the originals only after the new files are durable.

## Startup Cleanup Protocol

Any crash-safe scheme that uses temp files needs a cleanup step on startup:

```python
def _cleanup_temps(self):
    """Delete orphaned .tmp files from interrupted truncations."""
    for tmp in glob.glob(os.path.join(self._log_dir, "*.wal.tmp")):
        os.unlink(tmp)
```

This runs in the constructor before `_recover_seq_num()`. An orphaned `.tmp` file means the rename never completed, so the original `.wal` file is still intact and authoritative.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_record` — The CRC validation and partial-read handling in this function determines what "corrupt" vs "torn write" means during recovery, which directly affects whether a crash-interrupted temp file would be detected
- [function] `b-tree-storage-engine/btree.py:commit` — Compare the B-tree's all-or-nothing WAL truncation (fsync data file, then truncate WAL) with the standalone WAL's partial truncation — understand why one is crash-safe and the other isn't
- [general] `directory-fsync-semantics` — On Linux (ext4, XFS) and macOS (APFS), directory fsync has different guarantees; explore when `os.fsync(dir_fd)` is actually necessary for rename durability
- [file] `hash-index-storage/bitcask.py` — The compaction method has the same class of crash-safety bug (delete-then-rename without atomic swap); trace the exact window where data loss occurs
- [general] `wal-segment-deletion-ordering` — When truncation deletes entire WAL segments, the order of `unlink()` calls matters for crash safety; explore whether deleting oldest-first vs newest-first changes the recovery invariants

## Beliefs

- `wal-truncate-not-crash-safe` — `truncate()` in `write-ahead-log/wal.py` rewrites surviving records in place via `open(f, "wb")`, so a crash mid-rewrite permanently loses both the original and replacement data
- `btree-wal-truncate-is-safe` — The B-tree's WAL `commit()` only truncates after fsyncing the data file, making truncation idempotent — a crash mid-truncation simply replays the WAL again on recovery
- `atomic-rename-requires-same-filesystem` — `os.rename()` is atomic only when source and destination are on the same filesystem; crash-safe truncation must write the temp file in the same directory as the WAL
- `crash-safe-truncation-three-step` — The correct pattern is write-to-temp + fsync temp + rename over original + fsync directory, ensuring the old file is never destroyed until the replacement is fully durable on disk
- `orphaned-tmp-means-original-intact` — If a `.wal.tmp` file exists on startup, the atomic rename never completed, so the original `.wal` file is authoritative and the temp file should be deleted

