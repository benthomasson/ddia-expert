# Function: log_write in b-tree-storage-engine/btree.py

**Date:** 2026-05-29
**Time:** 08:23

## `WAL.log_write` — Append a Page Write to the Write-Ahead Log

### Purpose

`log_write` records a pending page write to the WAL file **before** the actual data file is modified. This is the core of the WAL protocol: if the process crashes after `log_write` but before (or during) the actual page write, `WAL.recover` can replay the logged entry to bring the data file back to a consistent state. Without this method, a crash mid-write could leave a half-written page in the data file — corrupting the B-tree.

### Contract

**Preconditions:**
- `self._f` is open and writable.
- `page_data` is a `bytes` object representing a complete page (callers always pad to `page_size` before calling — see `BTree._wal_write_page` at line 179).
- `page_num` is a valid page index (0 for metadata, ≥1 for data pages).

**Postconditions:**
- A complete WAL entry (header + data + checksum) is durably on disk (guaranteed by `fsync`).
- `self._seq` has been incremented by 1.
- The file cursor is at the end of the file.

**Invariant:** Entries in the WAL file are strictly ordered by monotonically increasing sequence numbers, with no gaps within a single session.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `page_num` | `int` | The page number in the data file this write targets. Packed as a 4-byte big-endian unsigned int. |
| `page_data` | `bytes` | The full page contents to be written. Length is encoded in the header, so variable-length data is supported at the WAL level (though callers always pass fixed `page_size` buffers). |

### Return Value

`None`. This is a side-effect-only method. The caller is responsible for subsequently calling `WAL.commit` to clear the log after the transaction completes.

### Algorithm

1. **Increment sequence counter** — `self._seq += 1`. This provides ordering and lets `recover` process entries in the order they were written.
2. **Pack the header** — 12 bytes total: sequence number (4B), page number (4B), data length (4B), all big-endian unsigned ints.
3. **Compute and pack checksum** — CRC32 of `page_data` only (not the header), stored as a 4-byte big-endian unsigned int. This detects torn writes during recovery.
4. **Seek to end of file** — `seek(0, 2)` positions the cursor at EOF so entries are appended, never overwriting earlier entries in the same transaction.
5. **Write the entry** — `header + page_data + checksum` as a single `write` call. The entry format is: `[seq:4][page_num:4][data_len:4][page_data:variable][crc32:4]`.
6. **Flush + fsync** — `flush()` pushes Python's userspace buffer to the OS, then `os.fsync()` forces the OS to flush to stable storage. This is what makes the durability guarantee real.

### Side Effects

- **Disk I/O**: Appends to the WAL file and forces a sync to stable storage. This is the most expensive part — `fsync` blocks until the hardware confirms persistence.
- **State mutation**: `self._seq` is incremented. This counter is not persisted independently; it resets to 0 when the WAL is committed or when a new `WAL` instance is created (it's reconstructed implicitly during recovery by reading entries).
- **File cursor**: Moved to end of file.

### Error Handling

None. If `struct.pack` fails (wrong types), `write` fails (disk full), or `fsync` fails (I/O error), the exception propagates uncaught. This is appropriate — a WAL write failure means the transaction cannot be made durable and must not proceed. The caller (`BTree._wal_write_page`) does not catch these either, so a failed WAL write aborts the entire `put`/`delete` operation.

### Usage Patterns

`log_write` is never called directly by external code. It's called by `BTree._wal_write_page` and `BTree._wal_write_meta`, which form a transaction bracket:

```python
# Typical transaction pattern (from BTree.put):
self.wal.log_write(page_num, padded)   # log intent
self.pm.write_page(page_num, padded)   # apply to data file
# ... possibly more log_write + write_page pairs ...
self.wal.commit(self.pm)               # clear WAL, transaction complete
```

A single B-tree mutation (e.g., a split) may call `log_write` multiple times before a single `commit`. All logged entries are replayed atomically during recovery — there is no partial replay.

### Dependencies

- **`struct`** — binary packing for the fixed-format header and checksum.
- **`zlib`** — via `_checksum` (CRC32). Used for integrity checking, not security.
- **`os.fsync`** — the POSIX durability primitive. This is what separates a WAL from a regular buffered log.

### Assumptions Not Enforced by Types

1. **`page_data` is bytes, not str** — `struct.pack('>I', self._checksum(page_data))` would fail on `str`, but the caller is responsible for encoding.
2. **Single-writer** — there's no file locking. If two processes open the same WAL, `_seq` values will collide and appended entries could interleave, corrupting the log.
3. **`write` is atomic for the payload size** — the code does a single `write(header + page_data + checksum)`. For typical 4KB pages, this is ~4KB + 16 bytes, which is generally atomic on local filesystems but not guaranteed by POSIX. A torn write here is detected by the checksum in `recover`, which skips corrupted entries.
4. **The checksum covers only `page_data`, not the header** — a corrupted header (wrong `data_len`) could cause `recover` to read the wrong number of bytes, potentially misaligning all subsequent entries. The recovery loop handles this by bailing out when `offset + data_len + 4 > len(data)`.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:WAL.recover` — The other half of the durability contract: replays logged entries and validates checksums after a crash
- [function] `b-tree-storage-engine/btree.py:WAL.commit` — Clears the WAL after a successful transaction; understanding when and why it truncates is key to the atomicity guarantee
- [function] `b-tree-storage-engine/btree.py:BTree._wal_write_page` — The caller that pads data and coordinates WAL logging with actual page writes
- [general] `fsync-durability-guarantees` — How `fsync` interacts with disk caches, battery-backed write caches, and different filesystems (ext4, ZFS, etc.) — the WAL's correctness depends on `fsync` actually reaching stable storage
- [file] `b-tree-storage-engine/test_btree.py` — Tests that exercise crash recovery scenarios to verify the WAL protocol holds under simulated failures

## Beliefs

- `wal-append-only-until-commit` — WAL entries are only appended, never overwritten; the file is truncated to zero only on `commit` or after successful `recover`
- `wal-checksum-covers-data-only` — The CRC32 checksum covers `page_data` but not the entry header, so header corruption is detected indirectly via length mismatch rather than checksum failure
- `wal-fsync-per-entry` — Each `log_write` call forces an `fsync`, guaranteeing durability of individual entries at the cost of one sync per logged page
- `wal-seq-monotonic-per-session` — The sequence counter increments from 0 on each `WAL` instantiation and resets on `commit`; it is not globally unique across sessions
- `wal-single-writer-assumed` — The WAL has no file locking; concurrent writers would corrupt the log with interleaved entries and duplicate sequence numbers

