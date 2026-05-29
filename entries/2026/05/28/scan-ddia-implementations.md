# Scan: ddia-implementations

**Date:** 2026-05-28
**Time:** 18:05

I don't have read access to the target repo, but the structure is highly informative. Here's my analysis based on the directory layout, file naming conventions, and deep knowledge of the DDIA concepts involved.

---

## Architecture Sketch

This is a **teaching-oriented reference implementation collection** — each top-level directory is a self-contained module implementing one concept from *Designing Data-Intensive Applications*. Every module follows a consistent three-file pattern: one implementation file, one test file, and often a `tester_test_*.py` (likely a meta-test or test harness validator). There are no shared libraries, framework code, or cross-module imports — each module is intentionally standalone so it can be understood in isolation. The implementations are in Python and appear to be pure algorithmic/protocol simulations rather than production systems.

---

## Module Map

The ~35 modules group naturally into the same layers as the book:

### Storage Engines & Data Structures (Part I: Foundations)
| Module | DDIA Chapter |
|--------|-------------|
| `hash-index-storage` / `log-structured-hash-table` | Ch 3 — Bitcask-style hash index |
| `sstable-and-compaction` | Ch 3 — Sorted String Tables |
| `log-structured-merge-tree` | Ch 3 — LSM-Tree |
| `b-tree-storage-engine` | Ch 3 — B-Tree |
| `write-ahead-log` | Ch 3 — WAL for crash recovery |
| `bloom-filter` | Ch 3 — Probabilistic membership |
| `avro-serializer` | Ch 4 — Schema evolution |

### Replication & Consensus (Part II: Distributed Data)
| Module | DDIA Chapter |
|--------|-------------|
| `leader-follower-replication` | Ch 5 — Single-leader |
| `multi-leader-replication` | Ch 5 — Multi-leader |
| `leaderless-replication` | Ch 5 — Dynamo-style |
| `read-repair` | Ch 5 — Anti-entropy |
| `hinted-handoff` | Ch 5 — Availability during partitions |
| `leader-election` | Ch 8 — Leader election |
| `raft-consensus` | Ch 9 — Raft |
| `two-phase-commit` | Ch 9 — 2PC |
| `total-order-broadcast` | Ch 9 — Total order broadcast |

### Partitioning (Part II)
| Module | DDIA Chapter |
|--------|-------------|
| `consistent-hashing` | Ch 6 — Hash partitioning |
| `range-partitioning` | Ch 6 — Range partitioning |
| `secondary-index-partitioning` | Ch 6 — Partitioned indexes |

### Transactions & Isolation (Part II)
| Module | DDIA Chapter |
|--------|-------------|
| `snapshot-isolation` | Ch 7 — MVCC |
| `write-skew-detection` | Ch 7 — SSI |
| `fencing-tokens` | Ch 8 — Fencing for distributed locks |

### Clocks & Ordering (Part II)
| Module | DDIA Chapter |
|--------|-------------|
| `lamport-clocks` | Ch 8 — Logical clocks |
| `vector-clocks` | Ch 8 — Vector clocks |

### Fault Tolerance (Part II)
| Module | DDIA Chapter |
|--------|-------------|
| `gossip-protocol` | Ch 5/8 — Failure detection |
| `byzantine-fault-tolerance` | Ch 8 — PBFT |
| `merkle-tree` | Ch 5 — Anti-entropy verification |

### Stream & Batch Processing (Part III: Derived Data)
| Module | DDIA Chapter |
|--------|-------------|
| `mapreduce-framework` | Ch 10 — MapReduce |
| `batch-word-count` | Ch 10 — Batch processing |
| `map-side-join` | Ch 10 — Join strategies |
| `change-data-capture` | Ch 11 — CDC |
| `event-sourcing-store` | Ch 11 — Event sourcing |
| `stream-join-processor` | Ch 11 — Stream joins |
| `partitioned-log` | Ch 11 — Log-based messaging |

### Integration (Part III)
| Module | DDIA Chapter |
|--------|-------------|
| `unbundled-database` | Ch 12 — Composing data systems |
| `conflict-free-replicated-data-types` | Ch 5/12 — CRDTs |

---

## Critical Files & Exploration Strategy

The exploration should follow the book's pedagogical arc: storage foundations first, then distribution, then derived data. Within each layer, start with the simpler concept and build toward the complex ones.

## Topics to Explore

### Layer 1: Storage Foundations (understand the data structures everything else builds on)

- [file] `write-ahead-log/wal.py` — WAL is the foundational crash-recovery primitive; replication, consensus, and event sourcing all depend on log-structured writes
- [file] `hash-index-storage/bitcask.py` — Simplest storage engine (Bitcask); establishes the append-only log + in-memory index pattern that LSM and SSTable extend
- [file] `log-structured-merge-tree/lsm.py` — LSM-Tree ties together memtable, SSTable, and compaction; the most architecturally rich storage engine in the collection
- [file] `b-tree-storage-engine/btree.py` — B-Tree is the counterpoint to LSM; understanding both reveals the read-vs-write tradeoff that drives all of Chapter 3

### Layer 2: Replication & Consensus (the hardest concepts; the core of distributed systems)

- [file] `leader-follower-replication/replication.py` — Single-leader replication is the baseline; establishes replication lag, failover, and log-based sync patterns
- [file] `leaderless-replication/dynamo.py` — Dynamo-style quorum reads/writes, sloppy quorums, conflict resolution — the alternative to leader-based models
- [file] `raft-consensus/raft.py` — Raft is the most complex protocol in the collection; understanding it unlocks leader election, log replication, and safety guarantees
- [file] `two-phase-commit/two_phase_commit.py` — 2PC shows how distributed transactions work (and why they're fragile); contrasts with consensus

### Layer 3: Transactions & Clocks (correctness guarantees)

- [file] `snapshot-isolation/mvcc_database.py` — MVCC is the dominant concurrency control mechanism; snapshot isolation shows how reads and writes coexist
- [file] `vector-clocks/vector_clock.py` — Vector clocks capture causality in distributed systems; they're the foundation for conflict detection in leaderless replication

### Layer 4: Partitioning & Fault Tolerance (scaling out)

- [file] `consistent-hashing/consistent_hashing.py` — Consistent hashing solves the partition assignment problem; used by Dynamo, Cassandra, and most distributed stores
- [file] `conflict-free-replicated-data-types/crdts.py` — CRDTs represent the state of the art in conflict-free replication; mathematically guaranteed convergence

### Layer 5: Stream & Batch (derived data systems)

- [file] `change-data-capture/cdc.py` — CDC bridges OLTP and derived data; it's the integration pattern that ties Part II to Part III
- [file] `unbundled-database/unbundled_database.py` — The capstone module; composes multiple concepts (log, CDC, indexes) into a unified data system, reflecting Chapter 12's thesis

### Cross-cutting

- [general] `tester-test-pattern` — Understanding the `tester_test_*.py` files that appear in most modules; likely a meta-testing or test validation harness that reveals how correctness is verified across all implementations

---

## Recommended Exploration Order

1. **WAL** then **Bitcask** — establish the log-append primitive
2. **LSM** then **B-Tree** — the two storage engine families
3. **Leader-follower** then **Leaderless** then **Raft** — replication spectrum
4. **2PC** then **MVCC** then **Vector clocks** — correctness mechanisms
5. **Consistent hashing** then **CRDTs** — partitioning and conflict resolution
6. **CDC** then **Unbundled database** — integration and capstone
7. **Tester pattern** — cross-cutting infrastructure understanding

This order mirrors the book's dependency chain: each concept builds on the ones before it, so beliefs extracted early will serve as antecedents for derived beliefs later.

