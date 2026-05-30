# Topic: Study how ETS tables provide concurrent read access with isolated write semantics, and why Python's GIL-protected `dict` is not equivalent (GIL doesn't help across processes)

**Date:** 2026-05-29
**Time:** 12:31

I have enough context to write the explanation. Here it is:

---

# ETS Tables vs. Python's GIL-Protected `dict`: Concurrency Models for Shared State

## The Core Question

When building a storage engine like Bitcask, the in-memory index (the "keydir") must be accessible to multiple concurrent clients. Erlang's ETS tables and Python's `dict` both serve as hash tables, but they occupy fundamentally different positions in the concurrency design space. Understanding why requires looking at the process models of each runtime.

## ETS: Shared-Memory Concurrency Across Processes

ETS (Erlang Term Storage) is a built-in concurrent hash table in the BEAM VM. Its key properties, as described in `topic-bitcask-paper-comparison.md:14`:

> The keydir is stored in an Erlang ETS table — a shared-memory concurrent hash table built into the BEAM VM. Multiple Erlang processes can read the keydir simultaneously without copying it, and writers perform atomic updates.

ETS provides three guarantees that make it suitable as a shared keydir:

1. **Atomic per-key updates** — A single `ets:insert/2` call atomically replaces one row. Multiple processes writing different keys concurrently never corrupt the table. Multiple processes writing the *same* key see a serialized, last-writer-wins result.

2. **Zero-copy reads across processes** — All BEAM processes on a node reference the same physical memory for an ETS table. No serialization, no message passing, no copying. From `topic-bitcask-parallel-hint-loading.md:52`: "Processes referenced the same physical memory, unlike Python's `dict` which is scoped to a single process."

3. **Named lookup** — Any process on the node can find the table by name via `ets:lookup/2`. This is what enables Riak vnodes to share a single keydir: when a new process opens an already-open Bitcask, it gets a reference to the *existing* in-memory keydir, not a fresh copy (`topic-bitcask-paper-comparison.md:14`).

This design is what made parallel hint file loading possible during startup. As documented in `topic-bitcask-startup-cost.md:121`:

> Riak's implementation (Erlang) used concurrent hint file loading across schedulers and merged results into the ETS keydir, which supports concurrent writes from multiple processes.

The BEAM VM runs one scheduler thread per CPU core. Bitcask could spawn one lightweight Erlang process per hint file, the scheduler distributed them across cores, and all processes inserted entries into the same ETS table concurrently without explicit locks (`topic-bitcask-parallel-hint-loading.md:36-41`).

## Python's `dict`: Process-Local, Thread-Unsafe (Despite the GIL)

Both Python Bitcask implementations use a plain `dict` for the keydir. From `topic-bitcask-paper-comparison.md:16-19`:

```python
self.keydir: dict[str, KeyEntry] = {}           # hash-index-storage/bitcask.py:35
self._index: dict[str, tuple[str, int]] = {}    # log-structured-hash-table/bitcask.py:44
```

There is no shared state between instances. The grep confirmation from `topic-bitcask-paper-comparison.md:21`:

> The grep for `shared|concurrent|thread|process|ETS` returns zero relevant hits. Each store instance owns its keydir exclusively.

### Why the GIL Doesn't Help

Python's Global Interpreter Lock (GIL) is often cited as making `dict` "thread-safe," but this is misleading in two critical ways:

**1. The GIL doesn't cross process boundaries.** Riak runs Bitcask across multiple Erlang processes — lightweight, isolated units of execution that share nothing by default. Python's equivalent for true parallelism is `multiprocessing`, where each OS process has its own interpreter and its own GIL. A `dict` in process A is completely invisible to process B. There is no equivalent of ETS's named lookup that lets process B say "give me a reference to process A's hash table." To share a keydir across Python processes, you'd need `multiprocessing.Manager`, shared memory with manual serialization, or an external store like Redis — all of which add copying, serialization overhead, or network hops that ETS avoids entirely.

**2. The GIL only protects interpreter internals, not application invariants.** Even within a single process using threads, the GIL guarantees that `dict.__setitem__` won't segfault, but it doesn't guarantee that a `check-then-act` sequence like:

```python
if key not in self.keydir:
    self.keydir[key] = entry  # Another thread may have inserted between check and set
```

is atomic. ETS's `ets:insert/2` is atomic at the row level by design — no external locking required.

### What This Costs the Python Implementations

The consequence is documented extensively across the knowledge base entries:

- **Every instance rebuilds from scratch.** From `topic-bitcask-paper-comparison.md:22-23`: "These Python implementations would need to reconstruct the entire keydir from disk on each instantiation (which they do — see `_rebuild_index` and `_recover`). For a keydir with millions of entries, this startup cost is significant — exactly the problem ETS sharing solves."

- **No parallel startup.** From `topic-bitcask-startup-cost.md:106-113`: Both implementations process files strictly sequentially in a `for fid in sorted(file_ids)` loop. No threading, no multiprocessing, no async I/O. This is constrained by the single-writer, no-synchronization architecture.

- **No concurrent access safety.** From `topic-bitcask-paper-comparison.md:32`: "Neither implementation has any locking mechanism." Two `BitcaskStore` instances writing to the same directory will corrupt data.

## The Architectural Difference in One Table

This table from `topic-bitcask-parallel-hint-loading.md:93-98` captures it precisely:

| Layer | Erlang/ETS | Python `dict` |
|-------|-----------|---------------|
| **Concurrency model** | BEAM schedulers distribute processes across cores | Single-threaded; GIL serializes threads within one OS process |
| **Shared state** | ETS: concurrent reads/writes, zero-copy, named lookup | `dict`: process-local, no sharing without serialization |
| **Correctness** | Atomic per-key updates; barrier + file ID ordering for consistency | No concurrent access; single-writer assumption |

## Why This Matters Beyond Bitcask

The ETS vs. `dict` distinction illustrates a broader architectural pattern: **the concurrency model of your runtime constrains what data structures can be shared and how.** The BEAM VM was designed around message-passing processes with ETS as an explicit shared-memory escape hatch. Python was designed around a single-threaded execution model with the GIL as an implementation detail, not a concurrency primitive. The Bitcask paper's design assumes the former; the Python implementations work around the latter by simply not attempting concurrency at all — which is a valid educational choice, but means they can never serve as a storage engine inside a multi-process server.

---

## Topics to Explore

- [file] `entries/2026/05/29/topic-bitcask-paper-comparison.md` — Full side-by-side comparison of Erlang-specific production features (ETS keydir, lock files, background merge) vs. Python simplifications
- [function] `hash-index-storage/bitcask.py:_rebuild_index` — The sequential startup loop that would be parallelized with ETS in Erlang; trace the for-loop to see exactly why shared state is required for parallelization
- [general] `beam-scheduler-binding` — How BEAM scheduler threads bind to CPU cores via `+sbt` flags, and how this affects cache locality during parallel ETS writes from hint file loaders
- [file] `entries/2026/05/29/topic-concurrent-merge-safety.md` — Analysis of what breaks when compaction and reads happen concurrently; demonstrates why Python's lack of shared-state primitives prevents background merge
- [general] `python-multiprocessing-shared-memory` — Study Python 3.8+'s `multiprocessing.shared_memory` module as the closest analog to ETS; compare the manual serialization burden against ETS's transparent term storage

## Beliefs

- `ets-is-zero-copy-across-processes` — ETS tables are shared physical memory within a BEAM node; multiple Erlang processes reference the same hash table without serialization or copying, unlike Python's `dict` which is always process-local
- `gil-does-not-cross-process-boundaries` — Python's GIL serializes threads within a single OS process but provides no synchronization across `multiprocessing` processes, making it irrelevant for the multi-process access pattern that production Bitcask requires
- `python-bitcask-rebuilds-keydir-every-open` — Both Python implementations reconstruct the entire in-memory index from disk on each instantiation because there is no shared-memory mechanism to preserve it across instances
- `ets-insert-is-per-key-atomic` — A single `ets:insert/2` call atomically replaces one row without external locking, providing the correctness guarantee that enables concurrent hint file loading into a shared keydir

