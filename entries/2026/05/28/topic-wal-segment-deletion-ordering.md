# Topic: When truncation deletes entire WAL segments, the order of `unlink()` calls matters for crash safety; explore whether deleting oldest-first vs newest-first changes the recovery invariants

**Date:** 2026-05-28
**Time:** 18:54

# WAL Segment Deletion Order and Crash Safety

## The Implementation: Oldest-First Deletion

The `truncate` method (`write-ahead-log/wal.py:179`) iterates over segments returned by `_wal_files()` (`wal.py:82`), which returns files sorted lexicographically by filename. Since segments are named with zero-padded incrementing numbers (`{next_num:06d}.wal` at line 112), this produces **oldest-first** order.

The deletion path is:

```python
# wal.py:179-201 (reconstructed from observations)
def truncate(self, up_to_seq: int) -> None:
    with self._lock:
        # flush and close current fd
        ...
        for path in self._wal_files():       # oldest-first iteration
            kept = []
            with open(path, "rb") as f:
                while True:
                    rec = _read_record(f)
                    if rec is None: break
                    if rec.seq_num > up_to_seq:
                        kept.append(rec)
                if not kept:
                    os.remove(path)            # line 201: delete entire segment
```

## Why Oldest-First Is the Safe Order

**The recovery invariant**: `_recover_seq_num()` (`wal.py:87`) scans all surviving WAL files to find the highest sequence number. Recovery correctness requires that the surviving files form a contiguous suffix of the original sequence — there must be no "gaps" where a middle segment is missing but older and newer segments remain.

**Oldest-first deletion preserves this invariant.** If a crash occurs mid-truncation after deleting segments 1–3 but before deleting segment 4, the surviving files are {4, 5, 6, ...} — still a contiguous suffix. Recovery replays everything from segment 4 onward, which is correct because segments 1–3 contained only records with `seq_num <= up_to_seq`, meaning those records were already checkpointed to stable storage.

**Newest-first deletion would violate it.** If we deleted in reverse order (6, 5, 4, 3, 2, 1) and crashed after deleting segments 6 and 5, the surviving files would be {1, 2, 3, 4} — but these are exactly the segments we *intended* to discard. Recovery would replay already-checkpointed data while the un-checkpointed records in segments 5–6 would be permanently lost. This is data loss.

## A Subtlety: Mixed Segments

The truncate method also handles the case where a segment contains a mix of records below and above `up_to_seq`. In that case, `kept` is non-empty and the segment is rewritten (the code after line 201, not shown in the observations, presumably writes `kept` records back). This rewrite-in-place has its own crash safety concern — if the process crashes after truncating the file but before writing the kept records, those records are lost. A safer approach would be write-new-then-rename, but this implementation appears to take the simpler path.

## What's Missing From the Observations

The observations cut off at line 200, so I can't see:
1. The rewrite logic for mixed segments (what happens when `kept` is non-empty)
2. Whether the directory is `fsync`'d after deletions — without `fsync` on the directory, the `unlink` operations may not be durable on crash, meaning the "order" is moot at the filesystem level on some systems
3. Whether `_open_latest()` is called at the end of truncate to reopen the write fd

The absence of a directory `fsync` (no `os.fsync` on a directory fd appears anywhere in the grep results for `fsync`) is arguably a bigger crash safety gap than the deletion order — on Linux, file removal isn't guaranteed durable until the parent directory is fsynced.

## Topics to Explore

- [function] `write-ahead-log/wal.py:truncate` — Read lines 201+ to see the mixed-segment rewrite logic and whether directory fsync is performed
- [function] `unbundled-database/unbundled_database.py:truncate_before` — Compare this alternative WAL truncation implementation for different safety tradeoffs
- [general] `directory-fsync-after-unlink` — Investigate whether any implementation in the repo fsyncs the parent directory after file deletion, which is required for crash-durable unlinking on Linux
- [function] `write-ahead-log/wal.py:_recover_seq_num` — Trace what happens during recovery when segments have gaps, to verify the contiguous-suffix assumption
- [file] `log-structured-merge-tree/lsm.py` — The LSM tree calls `self._wal.truncate()` (line 314) with no sequence argument, suggesting a different truncation contract worth comparing

## Beliefs

- `wal-truncate-deletes-oldest-first` — `truncate()` iterates segments via `_wal_files()` which sorts by filename, producing oldest-first deletion order that preserves the contiguous-suffix recovery invariant
- `wal-files-sorted-lexicographic` — `_wal_files()` returns segments sorted by filename (`{N:06d}.wal`), which is equivalent to chronological order since segment numbers increment monotonically
- `wal-no-directory-fsync` — The WAL implementation never fsyncs the parent directory after segment creation or deletion, meaning file metadata changes (including unlinks) are not guaranteed durable on crash
- `wal-truncate-closes-write-fd` — `truncate()` flushes, fsyncs, and closes the current write file descriptor before iterating segments, preventing conflicts with the file it may need to delete or rewrite

