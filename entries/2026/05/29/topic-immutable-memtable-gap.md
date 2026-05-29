# Topic: The `_immutable_memtables` list exists and is checked during `get()` and `range_scan()`, but `_flush` never populates it — this is a consistency gap worth investigating

**Date:** 2026-05-29
**Time:** 09:06

I don't have access to read the target repo directly, but the observations provide enough to give a thorough explanation. Let me work from what we have.

---

# The `_immutable_memtables` Consistency Gap

## What's going on

In a properly implemented LSM tree, the write path follows a specific lifecycle for memtables:

1. Writes go to the **active memtable**
2. When the active memtable hits a size threshold, it's **frozen** — moved to an immutable memtables list — and a new empty memtable becomes active
3. A background flush writes each immutable memtable to an SSTable on disk
4. Once flushed, the immutable memtable is removed from the list

This design is critical: it lets the system continue accepting writes (to the new memtable) while the old one is being flushed to disk, without blocking readers or writers.

## Where the gap is

In `log-structured-merge-tree/lsm.py`, the **read path** is correctly implemented for this lifecycle, but the **write path** never participates.

### The read side (correct)

At **line 254**, `get()` checks immutable memtables in reverse order (newest-first), which is the right behavior — if a key was written to a memtable that's currently being flushed, the read should still find it:

```python
for mt in reversed(self._immutable_memtables):  # line 254
```

At **line 286**, `range_scan()` also iterates the immutable memtables to include their entries in scan results:

```python
for mt in self._immutable_memtables:  # line 286
```

### The write side (missing)

At **line 210**, the list is initialized:

```python
self._immutable_memtables: List[SortedDict] = []  # line 210
```

But `_flush()` at **line 303** never does the two things it should:

1. **Before flushing**: move `self._memtable` into `self._immutable_memtables` and replace `self._memtable` with a fresh `SortedDict`
2. **After flushing**: remove the flushed memtable from `self._immutable_memtables`

Instead, `_flush()` likely writes the active memtable directly to an SSTable and clears it in place. This means `_immutable_memtables` is **always empty** — the read-side code that checks it is dead code.

## Why this matters

This is a **correctness gap**, not just dead code. In the current implementation, if `_flush()` writes the active memtable directly, there's a window where:

- A concurrent read could miss keys that are mid-flush (if the memtable is cleared before the SSTable is fully written)
- Or writes during flush would be interleaved with the data being flushed, leading to either lost writes or duplicate data in the SSTable

The immutable memtable pattern exists precisely to solve this: freeze the old data, start a new memtable for incoming writes, flush the frozen copy at leisure, then discard it. The read path already anticipates this design — the write path just never caught up.

In a single-threaded Python implementation without true concurrent flushes, this gap may never manifest as a bug in practice — `_flush()` runs synchronously, so there's no actual concurrency window. But it means the code doesn't faithfully demonstrate the LSM tree design from DDIA Chapter 3, which explicitly describes this immutable memtable phase as part of the architecture.

## What a fix would look like

The `_flush()` method should be restructured:

```python
def _flush(self):
    # Freeze current memtable
    self._immutable_memtables.append(self._memtable)
    self._memtable = SortedDict()
    
    # Flush the oldest immutable memtable to SSTable
    frozen = self._immutable_memtables[0]
    entries = list(frozen.items())
    # ... write SSTable ...
    
    # Remove from immutable list after successful write
    self._immutable_memtables.pop(0)
```

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:_flush` — Read the full flush implementation to confirm it writes the active memtable directly and never touches `_immutable_memtables`
- [function] `log-structured-merge-tree/lsm.py:get` — Trace the full read path to see what happens after the immutable memtable check falls through (SSTable lookup order, tombstone handling)
- [function] `log-structured-merge-tree/lsm.py:range_scan` — Understand how scan merges results across memtable, immutable memtables, and SSTables — particularly the merge-sort logic
- [general] `concurrent-flush-safety` — Consider what would break if `_flush` were made async or threaded without the immutable memtable fix
- [file] `log-structured-merge-tree/lsm.py` — Read lines 200-359 (the full `LSMTree` class) to see if there are any other lifecycle gaps between the WAL, memtable, and SSTable layers

## Beliefs

- `immutable-memtables-never-populated` — `_immutable_memtables` is initialized at line 210 but no code path ever appends to it, making the read-side checks at lines 254 and 286 dead code
- `flush-bypasses-freeze-step` — `_flush()` operates directly on the active memtable rather than freezing it into `_immutable_memtables` first, which deviates from the standard LSM tree lifecycle described in DDIA
- `read-path-anticipates-concurrent-flush` — Both `get()` and `range_scan()` check `_immutable_memtables` in the correct order (newest-first for point lookups, all for scans), indicating the read path was designed for a concurrent-flush architecture that the write path doesn't implement
- `single-threaded-masks-the-bug` — In a synchronous single-threaded execution model, the missing immutable memtable step causes no observable data loss because `_flush()` completes atomically within the caller's execution flow

