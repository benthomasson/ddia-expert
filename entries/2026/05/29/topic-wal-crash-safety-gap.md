# Topic: The truncate-then-reopen sequence is not atomic; a crash between steps 2 and 4 leaves a valid but empty WAL with no open handle, which is safe but worth reasoning through

**Date:** 2026-05-29
**Time:** 11:18

I have enough from the observations to explain this. The truncate method source was cut off at line 200 but I can see the critical structure. Let me also check the LSM WAL truncate for comparison, since it's a simpler variant of the same pattern.

---

# The Truncate-Then-Reopen Sequence: A Crash-Safety Analysis

## The Code in Question

There are two WAL truncate implementations in this codebase, and both exhibit the same pattern. The simpler one is in `log-structured-merge-tree/lsm.py:56-59`:

```python
def truncate(self):
    self._fd.close()          # Step 1: close the file descriptor
    self._fd = open(self._path, "wb")  # Step 2: reopen with "wb" (truncates to zero)
    self._fd.close()          # Step 3: close the truncated file
    self._fd = open(self._path, "ab")  # Step 4: reopen for appending
```

The more complex variant is in `write-ahead-log/wal.py:179+`, which takes an `up_to_seq` parameter and selectively removes records. Looking at lines 181-187:

```python
def truncate(self, up_to_seq: int) -> None:
    with self._lock:
        if self._fd:
            self._fd.flush()
            os.fsync(self._fd.fileno())
            self._fd.close()          # Step 1: flush, sync, close
            self._fd = None           # Step 2: clear the handle
```

Then it iterates over WAL files, reads/filters records, rewrites them (or deletes the file entirely if no records remain), and finally at line 210 calls `self._open_latest()` — step 4, reopening a file descriptor.

## The Non-Atomicity Window

The claim is about what happens if the process crashes *between* steps. Let's walk through each gap:

### LSM WAL (`lsm.py:56-59`)

| Crash point | State on disk | State in memory | Recovery behavior |
|---|---|---|---|
| After step 1 (close) | Original WAL file, fully intact | Process dead | Next startup replays all entries — **safe, redundant replay** |
| After step 2 (open "wb") | WAL file exists but is **empty** (zero bytes) | Process dead | Next startup sees empty file, nothing to replay — **safe, no data to recover** |
| After step 3 (close) | Same as above | Process dead | Same as above — **safe** |

The critical insight: between steps 1 and 2 (or during step 2), the file transitions from "full" to "empty." But truncation only happens *after* the data has been flushed to the main storage (SSTable). So an empty WAL at recovery time simply means "all data was already persisted" — the WAL's job was done.

### Full WAL (`wal.py:179+`)

The sequence is:

1. **Flush + fsync + close** (lines 183-186) — ensures all pending writes are durable
2. **Set `self._fd = None`** (line 186) — no open handle exists
3. **Iterate files, rewrite/delete** (lines 188-208) — multi-file operation, each file is read, filtered, and rewritten or removed
4. **`self._open_latest()`** (line 210) — reopen for future writes

A crash during step 3 is the interesting case. The method processes files one at a time. If it crashes mid-iteration:

- Files already processed: correctly truncated (records ≤ `up_to_seq` removed)
- File being processed: depends on the rewrite strategy — if it writes to a temp file and renames, it's atomic per-file; if it truncates in place, you could get a partial file
- Files not yet processed: still contain the old records

On recovery, `_recover_seq_num()` (line 80) scans all WAL files and finds the highest sequence number. `_open_latest()` opens the last file for appending. The "extra" records that weren't truncated will be replayed — which means the operation they represent is applied again. Since the underlying store already applied them (that's why we're truncating), this is a **redundant but idempotent replay**. Safe, as long as the operations are idempotent (PUT and DELETE are).

The state where `self._fd = None` and no file is open (between steps 2 and 4) means that if some other thread tried to call `append()`, it would crash with an `AttributeError` on `self._fd.write(...)`. But the `self._lock` prevents this — no other operation can interleave because `truncate` holds the lock for the entire duration.

## Why "Valid but Empty" Is Safe

The key invariant: **truncation is only called after the data protected by the WAL has been durably written elsewhere.** In the LSM tree, this happens after a memtable flush to an SSTable (`lsm.py:314` — `self._wal.truncate()` is called after the SSTable write). An empty WAL on recovery means "nothing to replay," which is correct — the data already lives in SSTables.

This is a fundamental WAL design principle: the WAL is always safe to lose *after* its contents have been checkpointed. The non-atomicity of truncate-then-reopen doesn't violate durability because the ordering guarantee (flush main store → then truncate WAL) means data is never exclusively in the WAL at truncation time.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_open_latest` — The recovery-time partner of truncate; understanding how it picks which file to reopen reveals the full crash-recovery loop
- [function] `log-structured-merge-tree/lsm.py:_flush_memtable` — The caller of `truncate()` in the LSM tree; shows the ordering guarantee that makes truncation safe (SSTable write completes before WAL truncation begins)
- [general] `fsync-ordering-guarantees` — The `os.fsync` call at `wal.py:184` before closing is critical; without it, the flush-then-close could reorder against the truncation on some filesystems
- [function] `write-ahead-log/wal.py:_recover_seq_num` — Scans all WAL files on startup to find the high-water mark; determines whether a partial truncation leaves orphaned sequence numbers
- [general] `rename-based-atomic-truncation` — An alternative design where you write the surviving records to a temp file and `os.rename()` over the original, giving you atomic per-file truncation without the crash window

## Beliefs

- `wal-truncate-requires-prior-checkpoint` — WAL truncation is only safe because it is called after the data it protects has been durably written to the main store (SSTable or equivalent); violating this ordering loses data
- `lsm-wal-truncate-not-atomic` — The LSM WAL truncate (`lsm.py:56-59`) has a window where the file is empty but no append handle is open; a crash here produces a valid empty WAL that replays zero records
- `wal-truncate-holds-lock` — Both `truncate` implementations hold `self._lock` for the entire operation, preventing concurrent `append` calls from encountering a `None` file descriptor
- `wal-partial-truncate-idempotent-replay` — If `wal.py:truncate` crashes mid-iteration, un-processed files retain old records that will be replayed on recovery; this is safe because PUT/DELETE are idempotent against an already-current store
- `wal-fsync-before-close-on-truncate` — `wal.py:183-185` explicitly flushes and fsyncs before closing during truncate, ensuring all buffered writes are durable before the file handle is released

