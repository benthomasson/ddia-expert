# Topic: Implementing a `SuperVersion` wrapper around the LSM tree's `_sstables` list with `__enter__`/`__exit__` ref counting would be a high-value addition

**Date:** 2026-05-29
**Time:** 09:59

I have enough from the observations to give a thorough explanation. Let me work with what's already gathered.

---

# Why a `SuperVersion` Wrapper Would Be High-Value

## The Problem: Unprotected Concurrent Access to `_sstables`

The LSM tree in `log-structured-merge-tree/lsm.py` stores its SSTable list as a plain mutable field:

```python
self._sstables: List[SSTable] = []  # newest last  (line 211)
```

Every read operation iterates this list directly — `get()` walks it in reverse at **line 259** (`for sst in reversed(self._sstables)`), and `range_scan()` walks it forward at **line 280** (`for sst in self._sstables`). Meanwhile, two write paths mutate this same list:

1. **Flush** appends a new SSTable at **line 313** (`self._sstables.append(sst)`)
2. **Compaction** wholesale-replaces it at **lines 350–351**:
   ```python
   old_sstables = self._sstables
   self._sstables = [new_sst]
   ```

There is **no synchronization** between readers and writers. If a reader is mid-iteration over `_sstables` when compaction replaces the list and deletes old SSTable files (**line 352**, `for sst in old_sstables`), the reader will attempt to open files that no longer exist. This is a classic read-after-free bug — except the "free" is an `os.remove()` on the SSTable file.

## What a SuperVersion Solves

The term "SuperVersion" comes from RocksDB, where it wraps together the current memtable, the immutable memtable(s), and the current SSTable list into a single versioned snapshot. The key idea:

1. **Snapshot isolation for readers**: A reader grabs a reference to the current `SuperVersion` on entry, and that reference keeps the SSTable list (and its files) alive for the duration of the read — even if compaction creates a new version.

2. **Ref-counted cleanup**: Old SSTable files are only deleted when the last reader using that version releases its reference. No reader ever sees a half-replaced list or a deleted file.

3. **Context manager protocol**: Python's `__enter__`/`__exit__` makes this ergonomic:

```python
def get(self, key: str):
    with self._current_version() as sv:
        # sv.sstables is frozen — compaction can't pull it out from under us
        for sst in reversed(sv.sstables):
            found, val = sst.get(key)
            if found:
                return val
```

## Why the Current Code Is Fragile

Look at compaction in detail. At **line 316**, the threshold check fires:

```python
if len(self._sstables) >= self._compaction_threshold:
```

Then `compact()` merges all SSTables into one (**lines 321–351**) and deletes the old files (**line 352**). Between lines 350 and 352, there's a window where:

- `self._sstables` has been reassigned to `[new_sst]`
- But a reader that captured `self._sstables` before line 350 still holds the old list
- The files backing that old list are about to be deleted

In a single-threaded context this doesn't bite because Python won't interleave execution within a single thread mid-loop. But this is a **reference implementation for DDIA concepts** — the whole point is to demonstrate real storage engine patterns. Any consumer who adds threading, async I/O, or even just studies this as a model for building their own engine will hit this race immediately.

## What's Notably Absent

The codebase has **zero** `__enter__`/`__exit__` definitions in the LSM module (the grep for context manager patterns found them only in `write-ahead-log/wal.py` and virtualenv internals). There are **zero** references to `SuperVersion` or `super_version` anywhere. And while the WAL module in `write-ahead-log/wal.py` does use `self._lock` for thread safety (**lines 143, 155, 171, 181, 217, 247, 258**), the LSM tree has no locking at all.

The `sstable-and-compaction/sstable.py` module takes a slightly more defensive approach — its `CompactionManager` exposes `get_sstables()` at **line 315** which returns `list(self._sstables)` (a shallow copy). This prevents mutation-during-iteration but still doesn't prevent file deletion during reads.

## The Implementation Shape

A minimal `SuperVersion` for this codebase would:

```python
class SuperVersion:
    def __init__(self, memtable, sstables):
        self.memtable = memtable  # snapshot of current SortedDict
        self.sstables = list(sstables)  # frozen copy
        self._ref_count = 0

    def __enter__(self):
        self._ref_count += 1
        return self

    def __exit__(self, *args):
        self._ref_count -= 1
        if self._ref_count == 0:
            self._cleanup_if_superseded()
```

Compaction would create a **new** `SuperVersion` rather than mutating `_sstables` in place. Old files get deleted only when the old version's ref count drops to zero. This is the same pattern DDIA describes for MVCC — readers don't block writers, writers don't invalidate readers — applied to the storage engine's own metadata rather than user data.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:compact` — The compaction method where the race condition manifests; lines 321–352 show the unprotected swap-and-delete sequence
- [file] `write-ahead-log/wal.py` — The only module in this repo using `self._lock` for thread safety; compare its discipline with the LSM tree's lack thereof
- [file] `sstable-and-compaction/sstable.py` — The `CompactionManager` class (line 301+) takes a different approach to SSTable lifecycle management; `get_sstables()` returns a copy but doesn't solve the file-deletion race
- [general] `rocksdb-superversion-design` — RocksDB's `SuperVersion` struct in `db/column_family.h` is the canonical implementation of this pattern; studying it reveals how ref counting interacts with flush, compaction, and iterator pinning
- [general] `mvcc-for-metadata-vs-data` — DDIA discusses MVCC for user-visible data, but the same principle applies to engine-internal structures; this is the conceptual bridge between Chapter 7 (transactions) and Chapter 3 (storage engines)

## Beliefs

- `lsm-sstables-unprotected-mutation` — `LSMTree._sstables` is mutated by both `_flush()` (append) and `compact()` (replace) with no synchronization or versioning, making concurrent reads unsafe
- `compaction-deletes-before-reader-release` — `compact()` deletes old SSTable files immediately after replacing `self._sstables`, with no mechanism to defer deletion until active readers finish
- `sstable-compaction-manager-copies-list` — `CompactionManager.get_sstables()` returns a shallow copy of `_sstables`, preventing iterator invalidation but not file-deletion races
- `wal-module-uses-locking-lsm-does-not` — The WAL module in `write-ahead-log/wal.py` uses `self._lock` for thread safety while the LSM tree module has no locking or concurrency control

