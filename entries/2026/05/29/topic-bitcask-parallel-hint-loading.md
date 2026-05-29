# Topic: How Riak's Erlang Bitcask used ETS tables and concurrent schedulers to parallelize hint file loading across cores during startup

**Date:** 2026-05-29
**Time:** 08:47

I have enough from the knowledge base entries to provide a thorough explanation. Here's the analysis:

---

# How Riak's Erlang Bitcask Parallelized Hint File Loading at Startup

## The Problem: O(n) Startup Cost

Bitcask's design requires rebuilding the entire in-memory keydir (a hash table mapping every key to its on-disk location) at startup. Without optimization, this means scanning every byte of every data file — O(total data size). Hint files reduce this to O(total key size) by storing only key metadata, but with millions of keys across hundreds of files, even this optimized path can take significant time on a single core.

## Why Hint Files Are Embarrassingly Parallel

Each hint file is **self-contained**: it describes exactly one data file's keys, with no references to other files. This independence is the key insight that enables parallelization. As documented in `entries/2026/05/29/topic-bitcask-startup-cost.md:117`:

> Each hint file is self-contained — it describes one data file's keys. Multiple hint files can be loaded concurrently, with results merged into the keydir using last-writer-wins on file ID ordering.

The Python reference implementations make the sequential nature explicit (`topic-bitcask-startup-cost.md:109`):

```python
for fid in sorted(file_ids):    # One at a time, in order
    ...
```

Both `_rebuild_index` in `hash-index-storage/bitcask.py` and `_recover` in `log-structured-hash-table/bitcask.py` walk files one by one, checking for a `.hint` file (fast path) or falling back to a full data scan (slow path).

## The Erlang Parallelization Strategy

Riak's production Bitcask exploited three BEAM VM features to parallelize this:

### 1. Concurrent Hint File Loading Across Schedulers

The BEAM VM runs one scheduler thread per CPU core (by default). Erlang processes — lightweight green threads — are distributed across these schedulers automatically. During startup, Bitcask could spawn one Erlang process per hint file, and the BEAM scheduler would distribute them across cores with no explicit thread management.

As stated in `topic-bitcask-startup-cost.md:121`:

> Riak's implementation (Erlang) used concurrent hint file loading across schedulers and merged results into the ETS (Erlang Term Storage) keydir, which supports concurrent writes from multiple processes.

### 2. ETS as a Concurrent Shared-Memory Keydir

The critical enabler was **ETS (Erlang Term Storage)** — a built-in shared-memory concurrent hash table in the BEAM VM. From `topic-bitcask-paper-comparison.md:13`:

> The keydir is stored in an Erlang ETS table — a shared-memory concurrent hash table built into the BEAM VM. Multiple Erlang processes can read the keydir simultaneously without copying it, and writers perform atomic updates.

ETS provided several properties essential for parallel loading:

- **Atomic per-key updates**: Multiple processes could insert entries concurrently without corrupting the table
- **No copying**: Processes referenced the same physical memory, unlike Python's `dict` which is scoped to a single process
- **Named lookup**: Any process could find the keydir by name via `ets:lookup/2`, enabling zero-cost sharing across Riak vnodes

The Python implementations use plain `dict` instances instead (`topic-bitcask-paper-comparison.md:17-18`):
```python
self.keydir: dict[str, KeyEntry] = {}           # hash-index-storage
self._index: dict[str, tuple[str, int]] = {}    # log-structured
```

### 3. Barrier Synchronization + Ordered Merge

Parallelism introduces a conflict resolution problem: if the same key exists in multiple hint files (because it was written, then overwritten, then compacted), which entry wins? The answer is **file ID ordering** — the entry from the highest-numbered file wins because it's the most recent.

The parallel strategy works in two phases:

1. **Fan-out**: Spawn one process per hint file. Each process reads its hint file independently and inserts entries into the ETS table. Since ETS supports concurrent writes, processes don't need to coordinate during the loading phase.

2. **Barrier**: Wait for all processes to complete before declaring the keydir consistent. This is the synchronization point — until every file has been processed, the keydir may contain stale entries that a later file would overwrite.

The ordering guarantee comes from the fact that files are processed in ascending ID order and last-writer-wins: if file 5 and file 12 both contain key `"user:42"`, file 12's entry must be the final one in the keydir. With concurrent loading, this means either:
- Processes insert with a "write-if-newer" check (compare file IDs atomically)
- Or all processes write freely, followed by a post-barrier reconciliation pass

### 4. The Slow Path Complication

The slow path (full data file scan when no hint file exists) is harder to parallelize because tombstones create cross-file dependencies. A tombstone in file N logically deletes a key from file N-1. From `topic-bitcask-startup-cost.md:119`:

> Data file scans modify the keydir as they go, and tombstones in file N can delete entries from file N-1. If you parallelize, you need a merge step that respects ordering — scan files independently, then apply results to the keydir in ID order.

This means the slow path requires scanning independently but applying results sequentially — parallelizing I/O but serializing the merge.

## What the Python Implementations Trade Away

The sequential design of both Python implementations (`topic-bitcask-startup-cost.md:106`) is constrained by their **single-writer, no-synchronization architecture** (`topic-bitcask-startup-cost.md:113`). There's no file locking, no concurrent read safety, and no shared state between instances. Each instance rebuilds the entire keydir from scratch on every open (`topic-bitcask-paper-comparison.md:22`).

The production gaps are summarized in a table at `topic-bitcask-startup-cost.md:136-140`, listing "No parallelism" with impact "Startup time scales linearly with file count" and fix "Parallel hint file loading + ordered merge."

## Summary

The Erlang Bitcask startup optimization was a three-layer design:

| Layer | Mechanism | Purpose |
|-------|-----------|---------|
| **Concurrency** | BEAM schedulers | Distribute hint file reads across CPU cores |
| **Shared state** | ETS tables | Concurrent keydir writes without copying or explicit locks |
| **Correctness** | Barrier + file ID ordering | Ensure last-writer-wins after all files loaded |

This turned an O(keys) sequential scan into an O(keys / cores) parallel scan, which for a Riak node with millions of keys across hundreds of data files, could reduce startup time from minutes to seconds.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:_load_hint_file` — The sequential hint file parser that reads the entire file into memory and walks it with a manual cursor; this is the unit of work that Erlang parallelized per-file
- [function] `log-structured-hash-table/bitcask.py:_recover` — The startup recovery loop showing the sequential hint-file-then-fallback pattern; contrast with how Riak would fan this out across schedulers
- [general] `erlang-ets-concurrency-model` — How ETS provides concurrent reads with isolated atomic writes, and why Python's GIL-protected `dict` is not equivalent (GIL doesn't help across OS processes)
- [general] `beam-scheduler-binding` — How BEAM scheduler threads can be bound to specific CPU cores via `+sbt` flags, and how this affects cache locality during parallel hint loading
- [file] `entries/2026/05/29/topic-bitcask-paper-comparison.md` — The full comparison of Erlang-specific production features (ETS keydir, lock files, background merge) vs. Python simplifications

## Beliefs

- `hint-files-are-embarrassingly-parallel` — Each hint file is self-contained (describes exactly one data file's keys), making concurrent loading across BEAM schedulers possible with no inter-file dependencies during the read phase
- `ets-enables-concurrent-keydir-writes` — ETS tables support atomic per-key updates from multiple Erlang processes without explicit locking, which is what allows parallel hint loaders to insert into a shared keydir concurrently
- `parallel-hint-loading-requires-barrier` — All hint file loading processes must complete before the keydir is declared consistent; file ID ordering with last-writer-wins resolves duplicate keys after the barrier
- `slow-path-parallelism-requires-ordered-merge` — Unlike hint files, full data file scans have cross-file tombstone dependencies, requiring results to be applied to the keydir in ascending file ID order even if scans run concurrently

