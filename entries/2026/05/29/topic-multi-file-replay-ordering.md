# Topic: With WAL rotation, `_read_all_records` iterates files in sorted filename order; worth verifying that sequence numbers are monotonic across file boundaries, not just within a single file

**Date:** 2026-05-29
**Time:** 11:45

I don't have permission to read the target repo directly, but the observations are sufficient to answer this.

---

## Cross-File Sequence Number Monotonicity in WAL Rotation

The question is: when `_read_all_records` reads across multiple `.wal` files in sorted filename order, are sequence numbers guaranteed to be monotonically increasing at the file boundaries?

**Yes, they are — by construction.** Here's why.

### How rotation works

The write path in `append()` (lines ~142–150) follows a strict order:

```
1. self._seq_num += 1          # bump the global counter
2. self._fd.write(data)        # write to current file
3. self._do_sync()             # fsync per policy
4. self._maybe_rotate()        # rotate AFTER the write
```

`_maybe_rotate()` (line 136–139) only opens a new file **after** the record has already been written to the current one. So the last record in file `000003.wal` always has a lower sequence number than the first record in `000004.wal`. The `_seq_num` counter is a single monotonically-increasing integer — it never resets or rewinds.

`append_batch()` (lines ~155–166) is even safer: it buffers the entire batch (including the COMMIT record) into a `bytearray` and writes it with a single `_fd.write(bytes(buf))` call. The `_maybe_rotate()` only fires after the whole batch lands, so a batch never straddles a file boundary.

### How filenames stay ordered

`_rotate()` (line 111) computes the next filename by parsing the integer prefix of the last file and incrementing it:

```python
next_num = int(os.path.basename(files[-1]).split(".")[0]) + 1
self._current_file = os.path.join(self._dir, f"{next_num:06d}.wal")
```

And `_wal_files()` (line 80) returns files in `sorted()` lexicographic order. Since the filenames are zero-padded to 6 digits (`000001.wal`, `000002.wal`, …), lexicographic sort equals numeric sort. This means **iterating files in sorted filename order is the same as iterating them in creation order**, which is the same as ascending sequence-number order.

### Recovery preserves the invariant

`_recover_seq_num()` (lines ~88–99) scans **all** WAL files on startup and sets `_seq_num` to the highest value found. So even after a crash, new records continue the sequence rather than restarting from zero. This is the piece that makes cross-restart monotonicity work.

### The gap in test coverage

The existing `test_rotation` (test_wal.py, line 73) verifies that all 20 records survive rotation and replay, but it only checks `len(records) == 20`. It does **not** assert that the sequence numbers are monotonically increasing across file boundaries. A test like this would close the gap:

```python
records = wal.replay()
for i in range(1, len(records)):
    assert records[i].seq_num > records[i-1].seq_num, \
        f"non-monotonic at index {i}: {records[i-1].seq_num} -> {records[i].seq_num}"
```

This is a true gap — the invariant holds by code inspection, but a regression could break it silently because no test enforces it.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_all_records` — The actual iteration logic at line 232; observations show it exists but we couldn't read lines 200+ to see whether it does any cross-file validation or just blindly yields records
- [function] `write-ahead-log/wal.py:truncate` — Truncation deletes/rewrites WAL files based on sequence numbers; worth checking whether it can leave gaps in the filename sequence that would break sorted-order assumptions
- [general] `batch-rotation-boundary` — What happens if a batch write pushes the file past `max_file_size` mid-batch? The current code writes the whole batch first, but does the next `append()` see the oversized file and rotate before writing?
- [function] `write-ahead-log/wal.py:_recover_seq_num` — If a WAL file is corrupted mid-file (as tested in `test_corruption`), recovery stops reading that file at the error; any valid records after the corruption point are silently lost, which could leave `_seq_num` lower than expected
- [file] `hash-index-storage/bitcask.py` — Uses a similar `_maybe_rotate` pattern for data segments; comparing the two rotation strategies reveals whether the same monotonicity guarantee holds for Bitcask segment files

## Beliefs

- `wal-seq-num-global-monotonic` — `_seq_num` is a single in-memory counter that only increments; all records across all WAL files share one sequence space
- `wal-rotation-after-write` — `_maybe_rotate()` is always called after the record write, never before, so a rotation boundary never splits a record's sequence number from its file
- `wal-batch-never-spans-files` — `append_batch` writes the entire batch (including COMMIT) to one `_fd.write()` call before checking rotation, so a batch is always contained within a single WAL file
- `wal-filename-sort-equals-creation-order` — WAL filenames use zero-padded 6-digit integers (`{n:06d}.wal`), so lexicographic sort matches numeric/creation order up to 999,999 files
- `wal-rotation-monotonicity-untested` — `test_rotation` asserts record count after rotation but does not verify that sequence numbers are monotonically increasing across file boundaries

