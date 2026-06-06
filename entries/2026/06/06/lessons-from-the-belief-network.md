# Lessons from the Belief Network

After accumulating 1,405 beliefs (1,224 premises, 181 derived) across 37 DDIA reference implementations, fixing 14 bugs driven by belief analysis, and observing cascade retractions collapse entire reasoning chains — several recurring patterns emerge. These aren't individual bugs but structural tendencies that appear independently across storage engines, consensus protocols, replication systems, and derived data infrastructure.

## 1. Integrity degrades along the data pipeline

Data starts with partial integrity protection and loses it as it moves downstream. The WAL has CRC checksums on record payloads, but even those exclude routing metadata (sequence numbers, operation types). By the time data reaches SSTables through compaction, integrity verification drops to zero — no checksums at all. Neither layer can recover from detected corruption.

This isn't accidental. The write path gets careful engineering attention because it's the hot path. Compaction, hint file generation, and read-side verification are "just moving data around" — but they're where corruption propagates silently.

**Antecedent beliefs:** `payload-only-crc-leaves-metadata-unprotected`, `sstable-layer-compounds-integrity-and-performance-deficiencies`, `integrity-checking-is-both-incomplete-and-unrecoverable`

**Practical lesson:** Verify checksums at every stage boundary where data changes format or location, not just at initial write. Our round-2 fix to validate CRC during Bitcask compaction and hint generation addressed exactly this gap.

## 2. Testing validates the wrong failure model

Every distributed protocol implementation — 2PC, Raft, PBFT, Total Order Broadcast — is tested under synchronous, deterministic message delivery. The safety properties these protocols claim to guarantee (partition tolerance, Byzantine fault tolerance, consensus under crash failures) only manifest under *asynchronous* failure modes — exactly the condition no test exercises.

This makes safety claims unfalsifiable: the test suite structurally cannot produce a counterexample. A passing test tells you the protocol works when messages arrive in order and on time, which was never in question.

**Antecedent beliefs:** `protocol-safety-validated-only-under-synchronous-model`, `2pc-safety-gaps-compound-under-synchronous-simulation`, `consensus-safety-unverified-across-optimization-variants`

**Practical lesson:** Distributed protocol tests need a chaos layer — message reordering, duplication, delay, and partition injection. Without it, "all tests pass" is vacuously true for the interesting failure modes.

## 3. Concurrency safety is assumed, never enforced

Core components silently assume single-threaded access: the B-tree's PageManager, the consistent hash ring, the garbage collector. No locks, no assertions, no `threading.Lock`, no documentation of the constraint. Range scans lack snapshot isolation or concurrent modification protection.

The assumption is invisible until violated. When it is violated, corruption happens on both read and write paths independently — concurrent reads can observe torn state even if writes are serialized.

**Antecedent beliefs:** `multiple-core-components-assume-single-thread-silently`, `range-scan-lacks-safety-guarantees`

**Practical lesson:** If a component requires single-threaded access, assert it (debug-mode thread ID check) or document it with a type-level constraint. Silent assumptions become silent bugs when usage patterns change.

## 4. Recovery undoes its own invariants

Transaction isolation in the SSI implementation composes two layers: MVCC visibility (which writes are visible to which transactions) and SSI conflict detection (which concurrent writes conflict). Both depend on volatile, monotonic counters. A crash does two things simultaneously: resurrects aborted writes (because abort is a status change, not a disk rollback) and breaks counter monotonicity (because counters aren't persisted). Recovery destroys the guarantees the normal path carefully maintains.

This pattern — where the recovery path violates invariants the normal path enforces — recurs across the codebase. The WAL recovery path originally ignored sequence numbers that the write path carefully maintained. B-tree metadata writes weren't fsynced even though data writes were.

**Antecedent beliefs:** `transaction-isolation-composes-two-invariant-layers`, `abort-is-status-change-not-disk-rollback`, `mvcc-counters-must-be-monotonic`

**Practical lesson:** Every invariant the normal path maintains must be explicitly preserved (or explicitly re-established) by the recovery path. If recovery can't restore an invariant, the invariant isn't real — it's a fair-weather guarantee.

## 5. Divergence accumulates faster than repair

In the replication layer, write-path correctness gaps compound: sloppy quorums count hinted handoffs toward the quorum threshold, sub-quorum configurations are silently accepted, and conflict resolution logic is split across modules (CRDT merge functions, last-writer-wins timestamps, custom merge callbacks) with no unified resolution model.

The repair mechanism — Merkle-tree-based anti-entropy — can detect divergence at the key-range level but cannot fully reconcile it, because tombstone semantics differ at every layer (TTL-based in CRDTs, explicit in Bitcask, status-flag in transactions). Divergence grows monotonically: writes create it faster than repair resolves it.

**Antecedent beliefs:** `distributed-writes-have-compounding-correctness-gaps`, `anti-entropy-detects-but-cannot-fully-resolve-divergence`, `tombstone-lifecycle-fragmented-across-modules`

**Practical lesson:** Repair mechanisms must understand the same semantics as write paths. If writes create divergence through five different mechanisms and repair understands three of them, the gap only grows.

## 6. Binary formats prevent evolutionary repair

The storage stack uses rigid binary formats throughout: WAL records, SSTable blocks, B-tree pages, hint files. None include version fields, block alignment markers, or extensibility mechanisms. This means known vulnerabilities (missing checksums, no metadata protection, no resync capability after corruption) cannot be fixed incrementally — any format change requires a full migration.

Combined with the absence of defense-in-depth (no input validation, no redundant integrity checks), the system has no path from its current state to a more robust one without breaking backward compatibility.

**Antecedent beliefs:** `binary-formats-rigid-across-entire-storage-stack`, `no-defense-in-depth-against-corruption`

**Practical lesson:** Include a version byte in binary formats from day one. The cost is one byte per record; the value is the ability to fix format-level bugs without a flag day migration.

## 7. Derived systems have a fragile AND-contract

Materialized views, secondary indexes, and search indexes all require two independent conditions for consistency: (1) explicit flush to make mutations visible to CDC consumers, and (2) old-value capture in change events to remove stale entries. Either condition failing silently produces stale reads — the derived system serves data that no longer reflects the source of truth.

Neither condition is enforced by the API. A caller can insert, update, and delete without flushing. The CDC layer can emit events without old values (it reconstructs them heuristically). The consistency contract is implicit and fragile.

**Antecedent beliefs:** `derived-system-consistency-requires-flush-and-old-values`, `cdc-event-semantics-depend-on-reconstruction-heuristics`

**Practical lesson:** Make consistency prerequisites explicit in the API. If flush is required for correctness, either auto-flush or return a "dirty" marker that the caller must resolve. Implicit contracts become forgotten contracts.

## The meta-lesson: bugs live at boundaries

The recurring theme across all seven patterns is that **gaps cluster at boundaries**: between the write path and the recovery path, between storage layers (WAL to SSTable), between the single-node and distributed layers, between normal operation and failure modes, between the source of truth and derived systems.

Each component works correctly in isolation. The bugs live in the composition — in the assumptions one layer makes about another that are never checked, in the invariants that hold during normal operation but not during recovery, in the guarantees that are tested under the wrong failure model.

This is why belief-driven analysis found bugs that unit tests missed: unit tests verify components, but the belief network reasons about *relationships between* components. The derived beliefs that went deepest (depth 5-7) were always cross-cutting — combining observations from storage, replication, and recovery into conclusions about system-level behavior. And the fixes that collapsed the most derived beliefs (retracting `delete-before-rename-ordering` collapsed 31 beliefs in one cascade) were always at boundaries too.

## By the numbers

- **1,405 total beliefs** (1,224 premises, 181 derived)
- **1,311 IN**, 94 OUT after two rounds of fixes
- **14 bugs fixed** across 9 implementations, driven by belief analysis
- **58 derived beliefs collapsed** via cascade retraction after fixes
- **Max network depth reduced** from 7 to 4
- Two beliefs that flipped from OUT to IN after fixes: `derived-systems-maintain-consistency-when-position-durable` and `wal-sequence-numbers-enable-ordered-recovery` — positive properties that were gated on the bugs we fixed
