# Topic: How BEAM scheduler threads can be bound to specific CPU cores via `+sbt` flags, and how this affects cache locality during parallel hint loading

**Date:** 2026-05-29
**Time:** 12:41

Now I have enough context. The existing knowledge base discusses the `beam-scheduler-binding` topic as a planned but not-yet-explored entry. Let me provide the explanation based on what's documented and what I know about BEAM internals.

---

# BEAM Scheduler Thread Binding (`+sbt`) and Cache Locality During Parallel Hint Loading

## Context: Where This Fits

The knowledge base has documented how Riak's Erlang Bitcask parallelized hint file loading at startup (`topic-bitcask-parallel-hint-loading.md:33-41`): spawn one lightweight Erlang process per hint file, let BEAM schedulers distribute them across cores, and merge results into a shared ETS keydir. The missing piece is what happens at the hardware level — specifically, how the BEAM VM maps its scheduler threads to physical CPU cores and why that mapping matters for performance.

## BEAM Schedulers: One Thread Per Core

By default, the BEAM VM starts one scheduler thread per available CPU core (`topic-bitcask-parallel-hint-loading.md:37`):

> The BEAM VM runs one scheduler thread per CPU core (by default). Erlang processes — lightweight green threads — are distributed across these schedulers automatically.

Each scheduler is an OS thread with its own run queue of Erlang processes. The VM handles process migration between schedulers for load balancing. But OS threads, by default, are free to be scheduled on *any* core by the OS kernel — the kernel may migrate a thread between cores based on load, thermal throttling, or power management.

## The `+sbt` Flag: Pinning Schedulers to Cores

The `+sbt` (scheduler bind type) flag controls how BEAM scheduler threads are bound to CPU topology. It's passed to the `erl` command or set in the `vm.args` configuration file:

```
erl +sbt db    # default_bind: bind schedulers respecting CPU topology
erl +sbt tnnps # thread_no_node_processor_spread
erl +sbt u     # unbound (default: OS decides placement)
```

The key bind types are:

| Flag | Meaning | Strategy |
|------|---------|----------|
| `u` | Unbound | OS decides which core runs each scheduler (default) |
| `db` | Default bind | BEAM picks a topology-aware binding |
| `tnnps` | Thread no-node processor spread | Spread schedulers across physical processors, then cores |
| `s` | Spread | Spread across all available hardware threads |
| `ts` | Thread spread | One scheduler per hardware thread, spread across cores |
| `ps` | Processor spread | Spread across physical processors first |
| `nnts` | No-node thread spread | Like `ts` but ignoring NUMA nodes |

When a scheduler is *bound*, the OS kernel pins that thread to a specific core (via `sched_setaffinity` on Linux or equivalent). The thread stays on that core permanently — no migration.

## Why Binding Matters: Cache Locality

Modern CPUs have a hierarchy of caches: L1 (per-core, ~32-64KB, ~1ns), L2 (per-core or shared pair, ~256KB-1MB, ~4ns), L3 (shared across cores, ~8-32MB, ~10-30ns), and main memory (~50-100ns). When a thread runs on a core, its working set — the data it reads and writes frequently — gets pulled into that core's L1/L2 caches.

**Without binding** (`+sbt u`): The OS can migrate a scheduler thread to a different core at any time. When this happens:
1. The thread's hot cache lines on the old core become useless (they'll be evicted eventually)
2. The new core's caches are cold — every memory access is a cache miss until the working set is reloaded
3. For NUMA systems, the new core may be on a different NUMA node, adding memory access latency

**With binding** (`+sbt db` or similar): Each scheduler stays on its assigned core. The thread's working set stays warm in L1/L2. Repeated operations on the same data — like iterating through a hint file's key entries and inserting them into an ETS table — benefit from temporal locality.

## Impact on Parallel Hint Loading

During Bitcask startup, as described in `topic-bitcask-parallel-hint-loading.md:66-69`:

> **Fan-out**: Spawn one process per hint file. Each process reads its hint file independently and inserts entries into the ETS table.

Each hint-loading Erlang process does two cache-intensive operations:

1. **Sequential read of a hint file**: Walking a binary buffer of serialized key records. This is a streaming access pattern — the data is read once, sequentially. CPU prefetchers handle this well regardless of binding, so cache locality from `+sbt` has minimal impact here.

2. **Random writes into the ETS hash table**: Each key entry is inserted into the shared ETS table via `ets:insert/2`. The ETS table is a hash table in memory — insertions touch random cache lines depending on the hash of each key. This is where binding matters:

   - **With binding**: A scheduler doing hint loading keeps its hash-table-related cache lines warm. The ETS internal metadata (bucket arrays, write locks per segment) stays in L1/L2. Repeated insertions by the same scheduler hit warm cache lines for the table's structural data, even though the *payload* cache lines are random.
   
   - **Without binding**: If the OS migrates the scheduler mid-hint-file, the ETS metadata cache lines go cold on the old core and must be reloaded on the new core. For a hint file with thousands of entries, this could happen multiple times, each migration costing microseconds in cache warm-up.

3. **NUMA considerations**: On multi-socket servers (common for Riak deployments), `+sbt tnnps` or `+sbt ps` spreads schedulers across physical processors. This matters because ETS tables are allocated in a single NUMA node's memory. Schedulers on remote NUMA nodes pay a cross-node memory access penalty for every ETS write. The optimal strategy depends on whether you want to minimize ETS access latency (pack schedulers near the ETS allocation) or maximize I/O bandwidth (spread across all sockets for parallel disk reads).

## The Trade-off

Scheduler binding is not universally beneficial:

- **Bound schedulers lose flexibility**: If one hint file is 10x larger than the others, its scheduler is pinned to one core while other cores go idle after finishing their smaller files. The BEAM's work-stealing between schedulers mitigates this (processes migrate, not threads), but binding adds friction to load rebalancing.

- **Spread vs. pack on NUMA**: Spreading across NUMA nodes maximizes aggregate memory bandwidth (good for parallel I/O) but increases latency for shared data structures like the ETS keydir. Packing onto one node minimizes ETS latency but reduces I/O parallelism.

For Riak's production use case, `+sbt db` was the common recommendation — let the BEAM pick a topology-aware binding that balances these concerns.

## What the Python Implementations Don't Have

None of this applies to the Python implementations. As documented in `topic-bitcask-parallel-hint-loading.md:85`:

> The sequential design of both Python implementations is constrained by their single-writer, no-synchronization architecture.

Python's `dict`-based keydir is process-local (`topic-erlang-ets-concurrency-model.md:53`), the GIL prevents true parallelism within a process, and `multiprocessing` would require serialization overhead that negates the cache locality benefits. The BEAM's scheduler binding is a feature of a runtime designed around lightweight process concurrency — it has no analog in CPython.

---

## Observations Gaps

The knowledge base entries discuss `+sbt` only by reference — the topic appears as a planned exploration item in `topic-bitcask-parallel-hint-loading.md:108` and `topic-erlang-ets-concurrency-model.md:94` but was never written. The explanation above draws on general BEAM VM knowledge rather than code observations in the target repository. To ground this fully, you would want:

- Riak's actual `vm.args` configuration showing production `+sbt` settings
- Bitcask source code (`bitcask_nifs.erl`) showing how hint loading processes are spawned
- Benchmarks showing the effect of `+sbt u` vs `+sbt db` on startup time with varying hint file counts

---

## Topics to Explore

- [general] `numa-aware-ets-allocation` — How ETS tables are allocated in NUMA memory and whether the BEAM places them near the creating scheduler's node or in interleaved memory
- [function] `hash-index-storage/bitcask.py:_load_hint_file` — The sequential hint file parser that is the unit of work Erlang parallelized per-file; trace the memory access pattern to see why cache locality matters for the write side
- [general] `beam-work-stealing-vs-thread-pinning` — How BEAM's process migration (work stealing) interacts with thread binding — processes move between pinned schedulers, giving both load balance and cache stability
- [file] `entries/2026/05/29/topic-bitcask-startup-cost.md` — The upstream entry documenting O(n) startup cost and the parallelization opportunity that scheduler binding optimizes
- [general] `erlang-dirty-schedulers` — How dirty schedulers (for long-running NIF calls like file I/O) interact with `+sbt` binding and whether hint file reads would use dirty or normal schedulers

## Beliefs

- `sbt-db-is-riak-default` — Riak production deployments typically used `+sbt db` (default bind) to let the BEAM assign schedulers to cores in a topology-aware manner, balancing cache locality against load distribution
- `scheduler-binding-benefits-ets-writes-more-than-file-reads` — Sequential hint file reads benefit minimally from core pinning (prefetchers handle streaming access), but random ETS hash table insertions benefit significantly because table metadata stays in L1/L2 cache
- `numa-creates-ets-access-asymmetry` — On multi-socket servers, schedulers on the NUMA node where the ETS table is allocated get lower-latency access than remote schedulers, creating a performance asymmetry during parallel hint loading
- `unbound-schedulers-cause-cache-migration-storms` — Without `+sbt` binding, OS thread migration during hint loading forces repeated L1/L2 cache warm-up for ETS metadata, adding microseconds of latency per migration event

