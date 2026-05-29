# Topic: The ARIES-style invariant that the log must be durable before the data file changes are considered committed, and how this code inverts that (data durable → log cleared) because it uses a redo-only WAL

**Date:** 2026-05-29
**Time:** 06:36

# The Redo-Only WAL: Why This Code Clears the Log *After* Syncing Data

## The Classical ARIES Rule

In ARIES (and most textbook WAL protocols), the fundamental invariant is:

> **The log record describing a change must be durable on disk before the changed data page is written to the data file.**

This is literally what "write-ahead" means — the log is *ahead* of the data. ARIES needs this because it supports both **redo** (replay committed changes after a crash) and **undo** (roll back uncommitted changes after a crash). The log is the single source of truth about what happened, and the data file is a cache that may be behind.

In ARIES, "commit" means: write a COMMIT record to the log, fsync the log. That's it. The dirty data pages can be written to disk whenever — minutes later, if you want. If you crash before they're written, recovery replays the log to redo them.

## How This Code Inverts It

The B-tree WAL in `b-tree-storage-engine/btree.py` does something subtly different. Look at the three key methods:

**Step 1 — Log the new page image before touching the data file** (`log_write`, ~line 140):

```python
def log_write(self, page_num, page_data):
    self._seq += 1
    header = struct.pack(self.ENTRY_HEADER, self._seq, page_num, len(page_data))
    checksum = struct.pack('>I', self._checksum(page_data))
    self._f.seek(0, 2)
    self._f.write(header + page_data + checksum)
    self._f.flush()
    os.fsync(self._f.fileno())  # WAL durable FIRST — this part is orthodox
```

This *is* write-ahead logging. The WAL entry hits disk before the B-tree data file is modified. So far, same as ARIES.

**Step 2 — "Commit" by syncing data, then destroying the log** (`commit`, ~line 148):

```python
def commit(self, page_manager):
    page_manager.sync()           # fsync the DATA FILE
    self._f.seek(0)
    self._f.truncate(0)           # DESTROY the WAL
    self._f.flush()
    os.fsync(self._f.fileno())    # make the truncation durable
    self._seq = 0
```

Here's the inversion. In ARIES, commit = "force the log." Here, commit = "force the data, then erase the log." The data file becoming durable is what *makes* the operation committed. The WAL is then garbage — it served its purpose and is discarded.

**Step 3 — Recovery replays and then also destroys the log** (`recover`, ~line 153):

```python
def recover(self, page_manager):
    # ... read all WAL entries ...
    if self._checksum(page_data) == checksum:
        page_manager.write_page(page_num, page_data)  # redo into data file
        recovered += 1
    page_manager.sync()           # make the redo durable
    self._f.seek(0)
    self._f.truncate(0)           # then clear the WAL
    # ...
```

Recovery replays page images into the data file, syncs, then clears the WAL — the same "data durable → log cleared" pattern.

## Why This Works (and Why There's No Undo)

This is a **redo-only, physical** WAL. Two properties make the inversion safe:

1. **Physical page images, not logical operations.** The WAL stores the complete *after-image* of each page (`page_data` is the entire page). Recovery doesn't need to interpret operations — it just stamps pages back into the data file. Compare this with the standalone WAL in `write-ahead-log/wal.py`, which stores logical operations (PUT/DELETE with keys and values at ~lines 1-15) and requires application-level replay logic.

2. **No concurrent transactions.** There's no need to undo uncommitted work from other transactions. The B-tree does its writes, logs them, then calls `commit()`. If a crash happens before `commit()`, recovery replays the WAL entries (redo) to complete the interrupted operation. If the crash happens after `page_manager.sync()` but before `truncate()`, the WAL entries are redundant — replaying them is idempotent because they're full page images.

The crash window analysis:

| Crash point | WAL state | Data state | Recovery action |
|---|---|---|---|
| After `log_write`, before data write | Has entries | Incomplete | Replay WAL → data consistent |
| After data write, before `sync()` | Has entries | Maybe partial | Replay WAL → data consistent |
| After `sync()`, before `truncate()` | Has entries | Complete | Replay WAL (idempotent, no harm) |
| After `truncate()` | Empty | Complete | Nothing to do |

There is no row in this table where you need undo. That's the whole trick.

## The Contrast in One Sentence

ARIES says "the log is the authority; data pages are a materialized cache you can reconstruct from the log." This code says "the data file is the authority; the WAL is a temporary safety net you throw away once the data file is consistent."

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:log_write` — Trace how B-tree operations (insert, split) call `log_write` before each `write_page` to see the write-ahead discipline in action
- [file] `write-ahead-log/wal.py` — Compare the standalone logical WAL (operation-level PUT/DELETE/COMMIT records) against the B-tree's physical WAL (full page images) and consider why each design fits its use case
- [function] `b-tree-storage-engine/btree.py:PageManager.sync` — The `flush()`/`fsync()` pair at ~line 113 is what `commit()` calls to make data durable; understand why both calls are needed (userspace buffer → kernel buffer → disk platter)
- [general] `undo-logging-and-steal-policy` — Why ARIES needs undo at all: the STEAL buffer management policy lets dirty pages from uncommitted transactions be written to disk, which this simpler design avoids entirely
- [function] `write-ahead-log/wal.py:append_batch` — The COMMIT record pattern (~line 155) shows the logical-WAL equivalent of atomicity: operations before COMMIT are tentative, enabling transaction-style replay semantics

## Beliefs

- `btree-wal-is-physical-redo-only` — The B-tree WAL in `btree.py` stores full after-image page data and supports only redo recovery; there is no undo capability
- `commit-means-data-sync-then-wal-truncate` — `WAL.commit()` fsyncs the data file first, then truncates the WAL to zero; the data file being durable is the commit point, not a log record
- `wal-replay-is-idempotent` — Because the WAL stores complete page images, replaying entries that were already applied to the data file produces identical results, making the crash window after `sync()` but before `truncate()` harmless
- `two-wal-designs-in-repo` — The standalone `wal.py` uses a logical WAL (keyed operations with COMMIT markers), while `btree.py` uses a physical WAL (raw page images); they solve different problems and have different recovery semantics

