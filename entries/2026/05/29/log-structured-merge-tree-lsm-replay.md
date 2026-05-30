# Function: replay in log-structured-merge-tree/lsm.py

**Date:** 2026-05-29
**Time:** 11:44

## `WAL.replay` — Write-Ahead Log Recovery

### Purpose

`replay` reconstructs the in-memory state of uncommitted writes after a crash or restart. It reads the WAL file from disk and deserializes every entry that was appended but not yet flushed to an SSTable. Without this, any writes that reached the WAL but hadn't triggered a memtable flush would be silently lost.

This is the recovery half of the WAL contract: `append` guarantees durability (write-before-acknowledge), and `replay` guarantees that durable data is restored into the memtable on startup.

### Contract

- **Precondition**: `self._path` is set (always true after `__init__`). The file may or may not exist.
- **Postcondition**: Returns all *complete* entries from the WAL. Partial/corrupt trailing entries are silently discarded — the method never raises on truncated data.
- **Invariant**: The returned entries are in write-order (the same order they were appended).

### Parameters

None beyond `self`. The WAL path was fixed at construction time.

### Return Value

`List[Tuple[str, bytes]]` — a list of `(key, value_bytes)` pairs. The caller (`LSMTree._replay_wal`) inserts each pair directly into the active memtable via `self._memtable[key] = value`. The caller does **not** need to handle errors — an empty list means either no WAL exists or it was already truncated.

Values are raw bytes, not decoded strings. This includes tombstones (`b""`), which the caller must treat as deletions. Replay does not distinguish puts from deletes — it just restores whatever was written.

### Algorithm

1. **Bail if no file**: If the WAL path doesn't exist on disk, return an empty list. This handles first-run or post-truncation states.

2. **Bulk read**: Slurp the entire file into memory as a single `bytes` object. This is a deliberate choice — the WAL is bounded by the memtable threshold (default 1000 entries), so it's small enough to fit in RAM.

3. **Sequential deserialization loop**: Walk through the byte buffer with a manual cursor (`pos`), decoding the same TLV (type-length-value) wire format that `append` writes:

   ```
   [4B key_len][key_bytes][4B value_len][value_bytes]
   ```

   Each field is big-endian unsigned 32-bit (`>I` format in `struct`).

4. **Boundary checks at every step**: Before reading each length prefix or payload, the method checks whether enough bytes remain. If not, it `break`s out of the loop — treating the remainder as a torn write. This is the crash-tolerance mechanism: if the process died mid-`append`, the partial trailing entry is simply ignored.

5. **Accumulate and return**: Complete entries are appended to the list and returned in order.

### Side Effects

- **Disk I/O**: Opens and reads the WAL file. The file handle is opened and closed within the method (separate from the `self._fd` used by `append`).
- **No mutations**: Does not modify the WAL file, the memtable, or any other state. It's a pure reader — the caller decides what to do with the entries.

### Error Handling

The method is intentionally lenient:

- **Missing file** → empty list (not an error).
- **Truncated entry** → silently drops the partial tail and returns everything before it.
- **No checksums**: There is no CRC or integrity verification. A corrupt-but-complete entry (e.g., a bit flip that produces a valid length prefix pointing to garbage) will be silently returned as data. This is a simplification — production WALs (RocksDB, LevelDB) include per-record checksums.
- **Encoding errors**: If a key contains invalid UTF-8, `decode("utf-8")` will raise `UnicodeDecodeError`. This is unguarded — the assumption is that all keys were written by `append`, which encodes from valid Python strings.

### Usage Patterns

Called exactly once, during `LSMTree.__init__`, via `_replay_wal`:

```python
def _replay_wal(self):
    for key, value in self._wal.replay():
        self._memtable[key] = value
```

This replays writes into the fresh memtable before the tree accepts any new operations. The replay-then-serve sequence is the standard recovery protocol for WAL-backed storage engines.

After a successful flush-to-SSTable, `WAL.truncate()` empties the file, so replay on the *next* startup won't re-apply already-persisted entries. The correctness of this relies on the flush happening atomically before the truncate — if the process crashes between flush and truncate, replay will re-insert entries that are already in an SSTable, which is safe because the memtable values shadow SSTable values during reads (newer wins).

### Dependencies

- `os.path.exists` — file existence check
- `struct.unpack` — big-endian integer deserialization (mirrors the `struct.pack` in `append`)
- The wire format is coupled to `WAL.append` — the two methods must agree on `>I` length prefixes and the `[klen][key][vlen][value]` layout

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:WAL.append` — The write side of the WAL protocol; understanding the wire format makes replay's parsing logic obvious
- [function] `log-structured-merge-tree/lsm.py:LSMTree._flush` — The flush-then-truncate sequence that determines when WAL entries become redundant
- [general] `wal-torn-write-recovery` — How production WALs (RocksDB, LevelDB) use checksums and record boundaries to distinguish torn writes from corruption, which this implementation skips
- [function] `log-structured-merge-tree/lsm.py:SSTable._read_entry` — Uses the same TLV wire format but reads from a file handle instead of a byte buffer; compare the two parsing approaches
- [general] `replay-idempotency` — Why re-applying WAL entries after a crash-between-flush-and-truncate is safe in an LSM tree (last-writer-wins semantics)

## Beliefs

- `wal-replay-tolerates-partial-writes` — `WAL.replay` silently discards any trailing partial entry, treating it as a torn write from a crash; it never raises on truncated data
- `wal-no-checksum-verification` — The WAL has no CRC or integrity check; a corrupt-but-length-valid entry will be returned as data without detection
- `wal-replay-order-matches-append-order` — Entries are returned in the same sequential order they were appended, which preserves last-write-wins semantics when replayed into a memtable
- `wal-replay-is-read-only` — `replay` does not modify the WAL file or any in-memory state; the caller (`_replay_wal`) is responsible for inserting entries into the memtable
- `wal-flush-truncate-replay-safety` — If a crash occurs between SSTable flush and WAL truncation, replay re-inserts already-persisted entries, which is safe because memtable values shadow SSTable values during reads

