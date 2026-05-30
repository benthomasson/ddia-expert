# Topic: The ordering guarantees between WAL fsync, data file fsync, and WAL truncation that make this protocol correct (ARIES-style redo logging)

**Date:** 2026-05-29
**Time:** 11:55

I have enough from the observations to explain this. The B-tree module has the clearest ARIES-style protocol; the other two WALs are interesting contrasts.

---

# WAL Fsync Ordering Guarantees: ARIES-Style Redo Logging

## The Core Protocol: B-tree WAL (`btree.py`)

The B-tree's `WAL` class (lines 120–175) implements a textbook three-phase commit protocol. The ordering between phases is the entire correctness argument.

### Phase 1: Log before write (WAL fsync)

```python
# btree.py:135–141
def log_write(self, page_num, page_data):
    self._seq += 1
    header = struct.pack(self.ENTRY_HEADER, self._seq, page_num, len(page_data))
    checksum = struct.pack('>I', self._checksum(page_data))
    self._f.seek(0, 2)
    self._f.write(header + page_data + checksum)
    self._f.flush()
    os.fsync(self._f.fileno())
```

Every page modification is written to the WAL **and fsynced to disk** before the corresponding data file page is updated. This is the "write-ahead" guarantee — the redo record is durable before the state it describes exists in the data file.

### Phase 2: Flush data, then truncate WAL (`WAL.commit`)

```python
# btree.py:143–149
def commit(self, page_manager):
    page_manager.sync()          # (A) fsync the data file
    self._f.seek(0)
    self._f.truncate(0)          # (B) truncate the WAL
    self._f.flush()
    os.fsync(self._f.fileno())   # (C) fsync the truncated WAL
    self._seq = 0
```

This is where the ordering is load-bearing. Three things happen **in strict sequence**:

| Step | Operation | What it ensures |
|------|-----------|-----------------|
| **(A)** | `page_manager.sync()` | All dirty pages are durable in the data file. `PageManager.sync()` (line 109) calls `flush()` + `os.fsync()`. |
| **(B)** | `truncate(0)` | WAL contents are discarded — but only **after** the data file is confirmed durable. |
| **(C)** | `os.fsync()` on the WAL | The fact that the WAL is empty is itself made durable. |

### Why this order is correct: crash analysis

The protocol is correct because **no matter when a crash occurs, recovery produces the right state**:

**Crash during Phase 1** (between `log_write` calls): The WAL contains a partial set of entries. Recovery replays whatever is there. Partial operations are fine — each WAL entry is a self-contained page image with a CRC checksum (`btree.py:166–167`). Entries with truncated writes or bad checksums are skipped.

**Crash after Phase 1, before (A)**: The WAL is fully fsynced. The data file may have none, some, or all of the page writes (they were written but not fsynced — the OS might have flushed some to disk). Recovery replays **all** WAL entries. This is safe because redo is **idempotent**: writing the same page data to the same page number produces the same result regardless of whether the page was already updated.

**Crash between (A) and (B)**: The data file is durable. The WAL still has entries. Recovery replays them again — redundant but harmless, because redo is idempotent.

**Crash between (B) and (C)**: The data file is durable. The WAL was truncated in the filesystem but the truncation may not be durable. Two outcomes:
- If the truncation reached disk: WAL is empty, no recovery needed. Correct.
- If the truncation didn't reach disk: WAL still has entries, they get replayed. Idempotent, so correct.

**Crash after (C)**: Clean state. Nothing to recover.

The key insight: **you can never lose data because the WAL is only discarded after the data file is confirmed durable, and replaying durable WAL entries is always idempotent.**

### Phase 3: Recovery (`WAL.recover`)

```python
# btree.py:151–170
def recover(self, page_manager):
    self._f.seek(0)
    data = self._f.read()
    if not data:
        return 0
    # ... parse entries, verify checksums ...
    if self._checksum(page_data) == checksum:
        page_manager.write_page(page_num, page_data)
        recovered += 1
    page_manager.sync()              # fsync replayed pages
    self._f.seek(0)
    self._f.truncate(0)              # clear WAL
    self._f.flush()
    os.fsync(self._f.fileno())       # fsync WAL clear
    return recovered
```

Recovery follows the same ordering: replay entries → fsync data file → truncate and fsync WAL. This means recovery itself is crash-safe — if you crash during recovery, you just recover again. The redo-idempotent property makes this safe.

## Contrast: The Standalone WAL (`wal.py`)

The standalone `WriteAheadLog` class (line 62+) is a **transaction log**, not an ARIES-style redo log. It doesn't manage a data file directly — it's a building block.

The `_do_sync` method (`wal.py:125–134`) shows three sync modes:
- **`sync`**: fsync after every append — strongest durability
- **`batch`**: fsync every N writes — trades durability for throughput
- **`none`**: no fsync — relies on OS to eventually flush

`append_batch` (`wal.py:148–163`) forces fsync regardless of mode, ensuring batch atomicity. The `truncate` method (`wal.py:179+`) also fsyncs before closing the file descriptor — preserving the invariant that the current WAL state is durable before any structural changes.

The difference: this WAL doesn't own a "commit" protocol. It provides the logging primitive; the consumer is responsible for the fsync ordering between the WAL and whatever data store it protects.

## Contrast: The LSM WAL (`lsm.py`)

The LSM tree's WAL (`lsm.py:13–60`) is notably **weaker**:

```python
# lsm.py:24–26
def append(self, key: str, value: bytes):
    ...
    self._fd.flush()    # flush to OS — but NO os.fsync()
```

It calls `flush()` but **never calls `os.fsync()`**. This means WAL entries may survive in the OS page cache but aren't guaranteed durable on disk. The `truncate()` method (`lsm.py:56–59`) reopens the file in `"wb"` mode to clear it — again with no fsync.

The LSM tree's flush-to-SSTable path (`lsm.py:303`, `_flush`) also calls `self._wal.truncate()` after writing an SSTable, but the ordering gap means: if the OS crashes between the SSTable write and the WAL truncate, the WAL entries could be replayed, producing duplicate SSTable entries. This is tolerable because the LSM merge process is itself idempotent — duplicate keys in different SSTables resolve by taking the newer sequence number — but it's a weaker guarantee than the B-tree's protocol.

## The Three Invariants

The ARIES-style protocol in this codebase relies on three invariants:

1. **Write-ahead**: WAL fsync completes before any data file mutation (`log_write` fsyncs before `write_page` is called)
2. **Commit ordering**: Data file fsync completes before WAL truncation (`commit` calls `sync()` then `truncate()`)
3. **Redo idempotence**: Replaying a WAL entry that was already applied produces the same state (physical page images, not logical operations)

Invariant 3 is what makes the protocol robust to crashes at any point — including crashes during recovery itself.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:PageManager` — The data file I/O layer; understanding how `write_page`/`sync` interact with the WAL's ordering guarantees is essential context
- [function] `log-structured-merge-tree/lsm.py:_flush` — The LSM memtable-to-SSTable flush path where WAL truncation happens without fsync, showing the weaker durability model
- [function] `write-ahead-log/wal.py:append_batch` — How batch atomicity is achieved via a COMMIT sentinel record and forced fsync, contrasted with single-record appends
- [general] `fsync-vs-flush-durability` — The difference between Python's `file.flush()` (pushes to OS kernel buffer) and `os.fsync()` (pushes to stable storage) is the crux of why the LSM WAL is weaker than the B-tree WAL
- [function] `b-tree-storage-engine/btree.py:WAL.recover` — Walk through the recovery path to see how CRC checksums handle partial writes and how the "recover then truncate" sequence is itself crash-safe

## Beliefs

- `btree-wal-fsync-before-data-write` — In `btree.py`, `WAL.log_write` calls `os.fsync` on the WAL file before the corresponding data page is written via `PageManager.write_page`, enforcing write-ahead ordering
- `btree-commit-syncs-data-before-wal-truncate` — `WAL.commit` calls `page_manager.sync()` (data file fsync) before `truncate(0)` on the WAL file, ensuring the data file is durable before redo records are discarded
- `lsm-wal-no-fsync` — The LSM tree's `WAL.append` (`lsm.py:24–26`) calls `flush()` but never `os.fsync()`, meaning its WAL entries are not guaranteed durable on power loss
- `btree-redo-is-idempotent` — The B-tree WAL stores physical page images (not logical operations), so replaying the same entry twice writes identical bytes to the same page offset, making recovery crash-safe and re-entrant
- `btree-recovery-is-crash-safe` — `WAL.recover` follows the same sync-then-truncate ordering as `WAL.commit`, so a crash during recovery simply triggers another identical recovery pass

