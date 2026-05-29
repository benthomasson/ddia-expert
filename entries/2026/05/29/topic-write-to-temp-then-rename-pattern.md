# Topic: None of the implementations use the atomic safe-save pattern (write temp, fsync, rename); explore whether compaction and SSTable creation should adopt it

**Date:** 2026-05-29
**Time:** 06:37

# The Missing Atomic Safe-Save Pattern

## What the Pattern Is

The atomic safe-save pattern is a well-known technique for crash-safe file creation:

1. Write to a temporary file in the same directory
2. `fsync()` the temporary file (flush to disk)
3. `rename()` the temp file to the final path (atomic on POSIX)

If a crash occurs during step 1 or 2, the final path either doesn't exist yet or still has its old content. You never see a half-written file at the target path.

## What the Implementations Actually Do

Every file-writing path in this codebase writes directly to the final destination. Let's trace each one:

### SSTable Creation (`sstable-and-compaction/sstable.py:49`)

`SSTableWriter.__init__` opens the final filepath immediately:

```python
self._f = open(filepath, "wb")
self._f.write(struct.pack(HEADER_FMT, MAGIC, VERSION, 0))
```

Then `finish()` (line ~85) writes the index, footer, seeks back to update the header entry count, and closes the file. **No fsync at all** — the data may still be in the OS page cache when `finish()` returns. A crash mid-write leaves a corrupt, partially-written SSTable at the final path.

### LSM SSTable Flush (`log-structured-merge-tree/lsm.py:80`)

```python
with open(path, "wb") as f:
```

Same pattern — writes directly to the final SSTable path. The `_flush()` method (line 303) writes the memtable contents here, then adds the SSTable to the live list. A crash during the write leaves an incomplete SSTable that the recovery code will try to open.

### Bitcask Compaction (`hash-index-storage/bitcask.py`)

The `compact()` method (visible from the structure) creates a new data file and writes merged entries. It also writes hint files directly (`_write_hint_file`, line 159):

```python
with open(self._hint_path(file_id), "wb") as f:
```

No fsync, no temp file. A crash during compaction can leave both the old and new data files in inconsistent states.

### Log-Structured Hash Table Compaction (`log-structured-hash-table/bitcask.py:264`)

```python
with open(compacted_path, "wb") as out:
```

Direct write to the compacted segment path. Then hint files at line 370:

```python
with open(hint_path, "wb") as hf:
```

Again, no temp-file indirection.

### WAL Truncation (`write-ahead-log/wal.py:203`)

```python
with open(path, "wb") as f:
```

This one is particularly interesting — the WAL `truncate()` method rewrites WAL files in place to remove old records. If the process crashes mid-rewrite, you lose both the old records AND the records that should have been kept. The WAL — the component whose entire purpose is crash recovery — is itself not crash-safe during truncation.

## The Contrast: What They Do Get Right

The WAL implementation (`write-ahead-log/wal.py`) does use `fsync` correctly for its **append path** — `_do_sync()` (around line 125) calls `os.fsync(self._fd.fileno())` after writes, and the Bitcask implementation (`hash-index-storage/bitcask.py:92`) has a `sync_writes` option that fsyncs after each record write. So the concept of durability is present; it's just not applied to the file-creation operations that are most vulnerable.

## Should Compaction and SSTable Creation Adopt It?

**Yes, unambiguously — especially for compaction.** Here's why:

### SSTable Creation (memtable flush)

When a memtable flushes to an SSTable, the WAL still holds the data. So in theory, a crash during flush can recover from the WAL. But:

- The LSM `_flush()` method deletes the WAL after creating the SSTable. If the SSTable is half-written and the WAL is already gone, you lose data.
- Even if the WAL survives, readers that discover a half-written SSTable file at the expected path will crash or return corrupt data instead of falling back to recovery.
- An atomic rename means: either the SSTable exists and is complete, or it doesn't exist. Recovery logic becomes trivial.

### Compaction

Compaction is the **most critical** place for atomic writes:

- Compaction deletes the input SSTables/segments after writing the output. If the output is half-written when a crash occurs, you've lost data permanently — the inputs are gone and the output is corrupt.
- There is no WAL protecting compaction — it operates on already-persisted data.
- The atomic pattern would be: write merged output to a temp file, fsync, rename to final path, **then** delete inputs. A crash at any point either leaves the old files intact or leaves both old and new files (detectable on recovery as a compaction that needs to resume or roll back).

### What the Fix Looks Like

```python
import tempfile

def atomic_write(final_path, write_fn):
    dir_name = os.path.dirname(final_path)
    fd, tmp_path = tempfile.mkstemp(dir=dir_name)
    try:
        with os.fdopen(fd, 'wb') as f:
            write_fn(f)
            f.flush()
            os.fsync(f.fileno())
        os.rename(tmp_path, final_path)
        # fsync the directory to persist the rename
        dir_fd = os.open(dir_name, os.O_RDONLY)
        os.fsync(dir_fd)
        os.close(dir_fd)
    except:
        os.unlink(tmp_path)
        raise
```

The directory fsync on the last line is often overlooked — without it, the rename metadata itself may not be persisted.

### The WAL Truncation Case

The `truncate()` in `write-ahead-log/wal.py` is the most dangerous omission. It rewrites WAL files to remove already-checkpointed records. This should absolutely use the temp-write-rename pattern, because losing a WAL file during truncation defeats the entire purpose of having a WAL. The current implementation at line 203 opens the file with `"wb"` which immediately truncates the existing content before writing the filtered records.

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:_flush` — Trace the full flush path: when the WAL is cleared relative to SSTable creation, and what happens if a crash occurs between the two
- [function] `write-ahead-log/wal.py:truncate` — The WAL truncation rewrites files in place; understand how this interacts with concurrent reads and crash scenarios
- [general] `compaction-input-deletion-ordering` — In both Bitcask implementations, identify when old segment files are deleted relative to the new compacted file being committed — this ordering determines whether data loss is possible
- [file] `hash-index-storage/bitcask.py` — The `compact()` method (past line 200) is where old data files are removed after merge; trace whether a crash between new-file-write and old-file-delete can leave the index inconsistent
- [general] `directory-fsync-after-rename` — Even with atomic rename, the directory entry must be fsynced for the rename to survive a crash; none of the implementations fsync directories

## Beliefs

- `no-atomic-file-creation` — No SSTable writer or compaction routine in any implementation uses the write-temp/fsync/rename pattern; all write directly to the final file path
- `wal-truncate-not-crash-safe` — `write-ahead-log/wal.py:truncate` rewrites WAL files in place with `open(path, "wb")`, making the WAL itself vulnerable to data loss during truncation
- `compaction-deletes-before-fsync` — Compaction routines delete input files after writing the merged output, but without fsyncing the output first, so a crash can lose both old and new data
- `fsync-used-only-for-appends` — `os.fsync` appears in the WAL append path and Bitcask record writes, but never in SSTable creation, compaction output, or hint file writes
- `no-directory-fsync-anywhere` — No implementation fsyncs the parent directory after creating or renaming files, meaning file creation metadata may not survive a crash even if file contents are fsynced

