# Function: recover in b-tree-storage-engine/btree.py

**Date:** 2026-05-29
**Time:** 08:12

## `WAL.recover` — Write-Ahead Log Crash Recovery

### Purpose

`recover` replays uncommitted WAL entries back into the data file after a crash. When the process dies between writing pages and calling `commit()`, the WAL file retains a record of intended writes. On the next startup, `recover` re-applies those writes to bring the data file to a consistent state. This is the core crash-safety mechanism — without it, a crash mid-operation leaves the B-tree with partially written pages.

### Contract

- **Precondition**: The WAL file is open and readable. `page_manager` must accept `write_page(page_num, data)` and `sync()` calls.
- **Postcondition**: All valid (checksum-verified) entries in the WAL have been replayed to the page manager. The WAL file is truncated to zero bytes and fsynced. The page manager's data file is fsynced.
- **Invariant**: Only entries with matching CRC32 checksums are applied — torn writes from a crash are silently skipped.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `page_manager` | `PageManager` | The page I/O layer that owns the data file. Receives replayed writes via `write_page` and is fsynced via `sync()`. |

### Return Value

Returns an `int` — the count of successfully recovered (replayed) entries. Returns `0` if the WAL was empty or contained no valid entries. The caller (`BTree.__init__`) doesn't act on this value, but it's useful for diagnostics.

### Algorithm

1. **Read the entire WAL** into memory (`self._f.seek(0); data = self._f.read()`). Early-return 0 if empty.

2. **Parse entries sequentially**. Each entry has the layout:

   ```
   [seq: 4B] [page_num: 4B] [data_len: 4B] [page_data: data_len B] [checksum: 4B]
   ```

3. **Bounds check** before each read:
   - Verify enough bytes remain for the 12-byte header (`offset + ENTRY_HEADER_SIZE <= len(data)`)
   - After reading the header, verify enough bytes remain for the payload + 4-byte checksum (`offset + data_len + 4 > len(data)`). If not, `break` — this is a torn write at the tail.

4. **Checksum verification**: Compute CRC32 of `page_data` and compare against the stored checksum. Only replay if they match. A mismatch means the entry was incompletely written during a crash — skip it silently.

5. **Replay**: Call `page_manager.write_page(page_num, page_data)` to overwrite the page in the data file.

6. **Finalize**:
   - `page_manager.sync()` — fsync the data file so all replayed writes hit disk.
   - Truncate the WAL to zero and fsync it — this is equivalent to `commit()`, marking recovery complete.

### Side Effects

- **Overwrites pages in the data file** — every valid WAL entry is written to the corresponding page number, whether or not the data file already had the correct data. This is idempotent: replaying the same WAL twice produces the same result.
- **Fsyncs the data file** via `page_manager.sync()`.
- **Truncates and fsyncs the WAL file** — after recovery, the WAL is empty.
- **Resets the file cursor** on both the WAL and (indirectly) the data file.

### Error Handling

There is no explicit error handling. If the WAL file is corrupt in a way that still parses (e.g., valid header pointing to wrong data), the checksum gate catches it. However:

- An I/O error during `write_page` or `sync` will propagate as an unhandled exception, leaving the WAL intact (truncation hasn't happened yet) — this is safe because the next startup will re-attempt recovery.
- The truncation happening *after* `sync()` is critical: if the process crashes after replaying but before truncating, the next startup replays again harmlessly (idempotent writes).

### Usage Patterns

Called exactly once during `BTree.__init__`:

```python
self.wal = WAL(wal_path)
self.wal.recover(self.pm)
```

This is unconditional — recovery runs on every startup. If the WAL is empty (normal shutdown called `commit()`), it returns 0 immediately. The caller doesn't need to check whether recovery is needed; the method handles the empty case.

### Dependencies

- `struct` — for unpacking binary WAL entry headers and checksums
- `zlib.crc32` — via `_checksum()`, used for integrity verification
- `os.fsync` — ensures WAL truncation is durable
- `PageManager.write_page` / `PageManager.sync` — the target for replayed writes

### Assumptions Not Enforced by Types

1. **Page data in the WAL matches `page_manager.page_size`**. `recover` writes whatever `data_len` says without checking it against the page manager's expected size. `PageManager.write_page` pads or truncates, but silently truncating recovered data would corrupt the page.
2. **WAL entries were written by the same code version** with the same `ENTRY_HEADER` format. There's no version or magic number in the WAL file.
3. **The sequence number (`seq`) is parsed but never used** — entries are replayed in file order regardless of sequence. If entries were somehow reordered on disk, the last write to a given page wins.
4. **No deduplication**: if the WAL contains two entries for the same page (e.g., metadata page 0 written multiple times in a transaction), both are replayed. The last one wins, which is correct only because WAL entries are appended in causal order.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:WAL.log_write` — How entries are appended to the WAL, including the fsync discipline that makes recovery possible
- [function] `b-tree-storage-engine/btree.py:WAL.commit` — The counterpart to recover: how the WAL is cleared after a successful operation, forming the commit boundary
- [function] `b-tree-storage-engine/btree.py:BTree._wal_write_page` — How the B-tree coordinates WAL logging with page writes during mutations
- [file] `write-ahead-log/test_wal.py` — Tests that likely exercise crash-recovery scenarios and validate the WAL protocol
- [general] `wal-crash-safety-ordering` — The ordering guarantees between WAL fsync, data file fsync, and WAL truncation that make this protocol correct (ARIES-style redo logging)

---

## Beliefs

- `wal-recover-checksum-gate` — WAL recovery only replays entries whose CRC32 checksum matches, silently skipping torn writes from crashes
- `wal-recover-then-truncate` — Recovery fsyncs the data file before truncating the WAL, ensuring replayed writes survive a crash during recovery itself
- `wal-recover-idempotent` — Replaying the WAL multiple times produces the same result because `write_page` overwrites unconditionally
- `wal-seq-unused-in-recovery` — The sequence number field is parsed during recovery but never consulted; replay order is determined solely by file offset
- `wal-recover-on-every-startup` — `BTree.__init__` calls `recover()` unconditionally; an empty WAL short-circuits with no I/O beyond the initial read

