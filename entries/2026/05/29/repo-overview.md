# Repo Overview

**Date:** 2026-05-29
**Time:** 11:39

Now I have a thorough understanding. Here's the overview:

---

# ddia-implementations: Repository Overview

## 1. Purpose

This repository is a **teaching-oriented collection of 37 standalone Python implementations** of core concepts from Martin Kleppmann's *Designing Data-Intensive Applications* (DDIA). Each module implements one data system concept — from B-trees and LSM-trees to Raft consensus and CRDTs — as a self-contained, runnable Python project. The goal is to make abstract distributed systems and storage engine concepts concrete and testable, not to provide production-ready infrastructure.

The implementations follow the book's pedagogical arc: storage engine foundations (Part I), distributed data and consensus (Part II), and derived data / stream processing (Part III).

## 2. Architecture

**One module per concept, zero cross-module coupling.** Every top-level directory is an independent implementation with no shared libraries, base classes, or cross-module imports. This is a deliberate architectural choice: each module can be understood, run, and tested in isolation without any context from the others.

The design pattern within each module is uniform:

```
<concept-name>/
├── <implementation>.py     # The concept implementation
├── test_<name>.py          # Pytest-based test suite (richer coverage)
└── tester_test_<name>.py   # Standalone test script (stdout-based validation)
```

All implementations use **Python standard library only** — no external dependencies beyond `pytest` for testing. This reinforces the teaching purpose: the algorithms and protocols are implemented from scratch, not wrapped around existing libraries.

## 3. Key Components

The 37 modules group into six layers that mirror the book's structure:

### Storage Engines & Data Structures (DDIA Part I, Chapters 3-4)

| Module | What it implements |
|--------|-------------------|
| `write-ahead-log/wal.py` | Length-prefixed, CRC-checked WAL with fsync modes (sync/batch/none), rotation, truncation, and crash replay |
| `hash-index-storage/bitcask.py` | Bitcask-style hash index — append-only log with in-memory keydir, hint files for fast startup, compaction |
| `log-structured-hash-table/bitcask.py` | Second Bitcask variant with different crash recovery semantics |
| `sstable-and-compaction/sstable.py` | Sorted String Tables with sparse index, size-tiered and leveled compaction strategies |
| `log-structured-merge-tree/lsm.py` | Full LSM-tree: memtable + SSTable flush + compaction + range scans |
| `b-tree-storage-engine/btree.py` | Disk-backed B-tree with page manager, WAL, node splitting, sibling leaf chains for range scans |
| `bloom-filter/bloom_filter.py` | Probabilistic set membership using Kirschner-Mitzenmacher double hashing |
| `avro-serializer/avro_serializer.py` | Avro binary encoding with schema registry and dual-schema resolution for evolution |

### Replication (DDIA Part II, Chapter 5)

| Module | What it implements |
|--------|-------------------|
| `leader-follower-replication/replication.py` | Single-leader with replication lag and failover |
| `multi-leader-replication/multi_leader.py` | Multi-leader with last-write-wins conflict resolution |
| `leaderless-replication/dynamo.py` | Dynamo-style quorum reads/writes, sloppy quorums |
| `read-repair/read_repair.py` | Anti-entropy read repair |
| `hinted-handoff/hinted_handoff.py` | Temporary writes during node unavailability |

### Consensus & Fault Tolerance (DDIA Part II, Chapters 8-9)

| Module | What it implements |
|--------|-------------------|
| `raft-consensus/raft.py` | Raft leader election, log replication, term-based safety |
| `two-phase-commit/two_phase_commit.py` | Coordinator-based 2PC with prepare/commit/abort |
| `total-order-broadcast/total_order_broadcast.py` | Total order broadcast protocol |
| `leader-election/leader_election.py` | Leader election primitives |
| `gossip-protocol/gossip_protocol.py` | Failure detection with heartbeat-based membership |
| `byzantine-fault-tolerance/pbft.py` | Practical Byzantine Fault Tolerance |
| `fencing-tokens/fencing_tokens.py` | Monotonic fencing tokens for distributed locks |

### Partitioning (DDIA Part II, Chapter 6)

| Module | What it implements |
|--------|-------------------|
| `consistent-hashing/consistent_hashing.py` | Consistent hash ring with virtual nodes and weighted placement |
| `range-partitioning/range_partitioning.py` | Range-based partition assignment |
| `secondary-index-partitioning/secondary_index_partitioning.py` | Document-partitioned and term-partitioned secondary indexes |

### Transactions & Ordering (DDIA Part II, Chapters 7-8)

| Module | What it implements |
|--------|-------------------|
| `snapshot-isolation/mvcc_database.py` | MVCC with append-only versions, garbage collection preserving active snapshots |
| `write-skew-detection/ssi_database.py` | Serializable Snapshot Isolation extending MVCC with dependency tracking |
| `lamport-clocks/lamport.py` | Lamport logical clocks |
| `vector-clocks/vector_clock.py` | Vector clocks with four-state comparison (BEFORE/AFTER/EQUAL/CONCURRENT) |
| `merkle-tree/merkle_tree.py` | Hash trees for anti-entropy verification |

### Stream & Batch Processing (DDIA Part III, Chapters 10-12)

| Module | What it implements |
|--------|-------------------|
| `mapreduce-framework/mapreduce.py` | MapReduce with map, shuffle, reduce phases |
| `batch-word-count/pipeline.py` | Batch processing pipeline |
| `map-side-join/map_side_joins.py` | Map-side join strategies |
| `change-data-capture/cdc.py` | CDC log with pull-based consumers, materialized views, search indexes, log compaction |
| `event-sourcing-store/event_store.py` | Event sourcing with live projections, snapshots, batch append |
| `stream-join-processor/stream_join_processor.py` | Stream join processing |
| `partitioned-log/partitioned_log.py` | Log-based messaging with partitions |
| `unbundled-database/unbundled_database.py` | Composing CDC, indexes, and logs into a unified data system (DDIA Ch 12 capstone) |
| `conflict-free-replicated-data-types/crdts.py` | G-Counter, PN-Counter, LWW-Register, OR-Set |

## 4. Data Flow

There is no system-level data flow — each module is self-contained. Within a module, the typical pattern is:

1. **Write path**: Client calls a mutation method (`put`, `insert`, `append`) → the implementation writes to its primary store (append-only log, WAL, in-memory structure) → durability is ensured via `fsync` (configurable) → secondary structures are updated (indexes, compaction triggers).

2. **Read path**: Client calls a read method (`get`, `select`, `range_scan`) → the implementation consults in-memory indexes or walks on-disk structures → returns the value or `None`/error.

3. **Recovery path** (storage engines): On startup, WAL or data files are scanned to rebuild in-memory state. For Bitcask, hint files accelerate this; for LSM, directory scanning finds SSTables; for B-tree, WAL entries are replayed into the data file.

4. **Derived data path** (CDC/event sourcing): Mutations append to an immutable log → downstream consumers poll the log from their cursor position → consumers build projections (materialized views, search indexes) from the event stream.

## 5. Dependencies

**Runtime: Python standard library only.** Every implementation uses only `struct`, `os`, `hashlib`, `typing`, `dataclasses`, `enum`, `threading`, `bisect`, and similar stdlib modules. No external packages are required.

**Testing: pytest.** The `test_*.py` files use pytest for assertion introspection and test discovery. The `tester_test_*.py` files are designed to run as standalone scripts (many still import pytest for convenience, but don't require it for execution).

This zero-dependency design is intentional — it forces the algorithms to be implemented from first principles rather than delegating to libraries.

## 6. Entry Points

There is no application entry point. Each module is a **library** consumed by its test files:

- **pytest**: Run `pytest` in any module directory (or at the repo root) to execute the test suite.
- **Standalone**: Run `python tester_test_<name>.py` in any module directory. These scripts print `"test_name PASSED"` / `"test_name FAILED"` to stdout and are designed for automated verification without pytest.

The `tester_test_*.py` files always include an `if __name__ == "__main__":` block that collects and runs test functions.

## 7. Configuration

Configuration is minimal and done entirely through constructor parameters — there are no config files, environment variables, or settings modules. Examples:

- **WAL**: `sync_mode` ("sync"/"batch"/"none"), `max_file_size` (rotation threshold), `batch_sync_count`
- **Bitcask**: `max_file_size`, `sync_writes` (defaults to `True` — fsync per record), `auto_compact_threshold`
- **LSM**: `memtable_threshold`, `compaction_threshold`
- **B-tree**: `page_size`, `order` (max keys per node)
- **Consistent hashing**: `num_vnodes`, per-node `weight`

## 8. Module Boundaries

**Strict isolation: no module imports from any other module.** Each directory is a completely independent implementation. The bloom filter exists as its own module but is not integrated into the LSM tree or SSTable modules — they coexist as parallel demonstrations of DDIA concepts, not as composable building blocks.

The one exception is conceptual layering *within* a module: `b-tree-storage-engine/btree.py` contains its own `WAL` and `PageManager` classes internally, rather than importing from `write-ahead-log/wal.py`. Similarly, `unbundled-database/unbundled_database.py` re-implements CDC concepts internally rather than importing from `change-data-capture/cdc.py`.

## 9. Extension Points

- **New DDIA concept**: Create a new top-level directory with the three-file pattern (`<impl>.py`, `test_<impl>.py`, `tester_test_<impl>.py`). No registration or configuration needed.
- **New compaction strategy**: `sstable-and-compaction/sstable.py`'s `CompactionManager` accepts a strategy string; add a new branch alongside `'size_tiered'` and `'leveled'`.
- **New CRDT type**: Add a new class to `crdts.py` following the existing pattern (state + merge + query).
- **New CDC consumer type**: Add a class alongside `MaterializedView` and `SearchIndex` in `cdc.py` that reads from `CDCLog`.

## 10. Known Constraints

- **Single-threaded assumption**: Most implementations have no concurrency control. LSM compaction, Bitcask compaction, and B-tree operations all assume a single writer. Where `threading.Lock` appears (WAL), it protects only basic append operations.
- **NO-STEAL buffer management**: All storage implementations keep uncommitted data in memory only — no undo logging, but all dirty data from active transactions must fit in RAM.
- **In-memory simulation**: Replication, consensus, and distributed protocols run as in-process simulations — nodes are objects in the same process, not networked services. Network partitions are simulated by filtering message delivery.
- **No persistence for distributed protocols**: Raft, 2PC, gossip, etc. maintain state in Python objects. There is no disk persistence or crash recovery for distributed protocol state.
- **macOS latent bugs**: Several implementations have `fsync`/directory-sync patterns that work correctly on macOS (APFS provides implicit rename durability) but would fail on Linux ext4/XFS. These are documented as latent bugs in the knowledge base.
- **CRC covers payload only**: The WAL, Bitcask, and B-tree all compute CRC32 over data payloads only, excluding headers — a consistent pattern that leaves header corruption undetected.

---

## Topics to Explore

- [file] `write-ahead-log/wal.py` — The foundational crash-recovery primitive; its binary record format, fsync modes, and truncation semantics establish patterns reused by the B-tree and Bitcask modules
- [file] `unbundled-database/unbundled_database.py` — The capstone module composing CDC, log-based messaging, and secondary indexes into a unified data system, reflecting DDIA Chapter 12's thesis about composing derived data
- [function] `b-tree-storage-engine/btree.py:_delete` — The delete path's three-valued return (`False`/`True`/`'empty'`) and its incomplete cleanup (only depth-2 parents, never leftmost child) reveals real-world tradeoffs in B-tree deletion
- [general] `crash-safety-gaps` — Multiple modules have documented crash-safety gaps (Bitcask compaction, B-tree page allocation, LSM compaction) where multi-step file mutations lack atomicity; comparing these reveals a consistent design choice to favor simplicity over production-grade durability
- [file] `raft-consensus/raft.py` — The most complex protocol implementation; understanding its term-based safety, leader election, and partition simulation illuminates why consensus is hard

---

## Beliefs

- `modules-are-fully-isolated` — No module imports from any other module; each top-level directory is a self-contained implementation with zero cross-module dependencies, including cases where integration would be natural (bloom filter + LSM tree)
- `all-implementations-stdlib-only` — Every implementation uses only Python standard library modules; no external runtime dependencies exist, enforcing first-principles implementation of all algorithms
- `dual-test-suite-convention` — Each module maintains two parallel test files: `test_*.py` (pytest, broader coverage) and `tester_test_*.py` (standalone scripts using stdout `PASSED`/`FAILED` protocol), with the tester files averaging ~60% the line count of their pytest counterparts
- `crc-covers-payload-not-headers` — The WAL, Bitcask, and B-tree storage engine all compute CRC32 over data payloads only, excluding header/framing metadata — a consistent design choice across the repo that leaves header corruption undetectable
- `no-steal-buffer-management` — All storage engine implementations use NO-STEAL policy: uncommitted transaction data never reaches disk, eliminating undo logging at the cost of requiring all active transaction state to fit in memory

