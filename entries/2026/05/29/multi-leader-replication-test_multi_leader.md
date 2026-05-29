# File: multi-leader-replication/test_multi_leader.py

**Date:** 2026-05-29
**Time:** 09:02

I'll work from the test file and the imports visible in it to explain the code.

---

# `multi-leader-replication/test_multi_leader.py`

## Purpose

This is the primary test suite for a multi-leader (multi-master) replication system — a reference implementation of the architecture described in DDIA Chapter 5. Each "leader" node independently accepts writes, and changes are asynchronously replicated to peers. The file validates that the core invariants of multi-leader replication hold: convergence after sync, correct conflict resolution, topology-aware propagation, tombstone handling, and clock monotonicity.

It doubles as both a pytest-compatible test suite and a standalone script (via the `__main__` block), making it useful for quick smoke tests during development.

## Key Components

### Test Functions

| Function | What it validates |
|---|---|
| `test_lww()` | Last-write-wins conflict resolution. Two DCs write the same key concurrently; after sync all nodes converge and a conflict is logged. |
| `test_custom_merge()` | Pluggable conflict resolution via `ConflictStrategy.CUSTOM_MERGE` with a user-supplied `merge_fn`. Demonstrates additive merge (counter semantics). |
| `test_ring()` | Ring topology propagation requires multiple sync rounds — at least 2 for a 3-node ring — before convergence. |
| `test_tombstone()` | Deletes propagate as tombstones. After sync, all nodes return `None` for the deleted key. |
| `test_lamport_clock()` | Writes on a single `ReplicaNode` produce strictly increasing Lamport timestamps. |
| `test_idempotency()` | Syncing the same state twice doesn't produce spurious conflicts or duplicate data. |
| `test_lww_tiebreak()` | When timestamps are equal, node ID serves as a deterministic tiebreaker — lexicographically greater ID wins (`'b' > 'a'`). |
| `test_conflict_logging()` | Conflicts are recorded with metadata including the key and the strategy that resolved them (`ConflictStrategy.LAST_WRITE_WINS`). |

### Imported Symbols (from `multi_leader`)

- **`MultiLeaderCluster`** — Orchestrates a cluster of replica nodes. Accepts a list of node IDs, an optional `topology` (default: fully connected), an optional `strategy`, and an optional `merge_fn`.
- **`ReplicaNode`** — A single leader node. Supports `put(key, value)` → timestamp, `get(key)` → value, `delete(key)`, and exposes a `conflict_log` list.
- **`ConflictStrategy`** — Enum with at least `LAST_WRITE_WINS` and `CUSTOM_MERGE`.
- **`Topology`** — Enum with at least `RING` (and implicitly a default fully-connected topology).

## Patterns

**Cluster-centric testing**: Tests construct a `MultiLeaderCluster`, issue writes to individual nodes via `cluster.node(id)`, then call `cluster.sync()` to simulate replication. This keeps the network topology and sync mechanics inside the cluster abstraction rather than manually wiring nodes.

**Convergence assertions via `all_converged()`**: Rather than checking every key on every node, tests use a cluster-level `all_converged()` predicate — a single point where "eventual consistency achieved" is defined.

**Deterministic conflict setup**: Each test writes the same key from different nodes *before* syncing, guaranteeing a conflict. Timestamps are implicitly sequential per-node (Lamport clocks starting from the same epoch), so concurrent writes from different nodes land on the same timestamp, exercising tiebreak logic.

**No fixtures or setup/teardown**: Each test is self-contained — creates its own cluster, does its work, asserts, prints. No shared state between tests.

## Dependencies

**Imports**: `multi_leader` (wildcard) — pulls in `MultiLeaderCluster`, `ReplicaNode`, `ConflictStrategy`, `Topology`, and likely a `ConflictRecord` or similar namedtuple for conflict log entries.

**Imported by**: `tester_test_multi_leader.py` likely runs or wraps these tests for automated grading/validation within the project's test harness.

## Flow

1. **Setup**: Construct a `MultiLeaderCluster` with N node IDs and optional config.
2. **Writes**: Issue `put()` / `delete()` calls to specific nodes. Each write returns a Lamport timestamp.
3. **Sync**: Call `cluster.sync()` to replicate changes across nodes according to the topology. For ring topology, `sync_until_converged()` loops sync rounds until `all_converged()` returns `True`, returning the round count.
4. **Assert convergence**: Check that all nodes agree on values (`all_converged()`) and that conflict metadata was recorded correctly.

Data transformation during sync (inferred): each node's write log is replicated to peers. On receipt, the receiver compares timestamps; if the incoming write conflicts with a local write (same key, different value), the configured strategy resolves it — either LWW with node-ID tiebreak, or a custom merge function. The loser is discarded (or merged), and a `ConflictRecord` is appended to `conflict_log`.

## Invariants

- **Convergence**: After sufficient sync rounds, `all_converged()` must return `True` — all nodes hold identical state.
- **Lamport monotonicity**: Successive `put()` calls on the same node yield strictly increasing timestamps (`ts1 < ts2 < ts3`).
- **LWW tiebreak determinism**: Equal timestamps resolve by node ID comparison; lexicographically greater ID wins.
- **Tombstone propagation**: A `delete()` on any node, once synced, causes `get()` to return `None` on all nodes.
- **Idempotent sync**: Syncing already-synchronized state produces no new conflicts and doesn't corrupt values.
- **Ring topology requires multi-round sync**: A 3-node ring needs at least 2 rounds (each hop covers one neighbor, not the full cluster).

## Error Handling

Tests use plain `assert` with descriptive messages — failures surface as `AssertionError` with context (e.g., `f'a got {cluster.node("a").get("counter")}'`). There's no exception handling; any unexpected error from the implementation propagates as a test failure. The `print()` calls at the end of each test act as progress markers when run via `__main__`.

## Topics to Explore

- [file] `multi-leader-replication/multi_leader.py` — The implementation: how `sync()` replicates the write log, how LWW and custom merge resolve conflicts, and how `all_converged()` is defined
- [function] `multi-leader-replication/multi_leader.py:ReplicaNode.put` — How Lamport timestamps are assigned and how the write log is structured
- [file] `multi-leader-replication/tester_test_multi_leader.py` — The automated test harness wrapper; may add network partition or failure injection scenarios
- [general] `conflict-resolution-strategies` — DDIA Chapter 5 covers LWW, custom merge, and CRDT-based approaches — compare this implementation's strategy enum against the book's taxonomy
- [general] `topology-propagation-bounds` — How many sync rounds different topologies (ring, star, fully-connected) need for N nodes to converge

## Beliefs

- `lww-tiebreak-uses-node-id` — When two writes have equal Lamport timestamps, the node with the lexicographically greater ID wins under LWW resolution
- `custom-merge-replaces-lww` — `ConflictStrategy.CUSTOM_MERGE` with a `merge_fn` overrides the default LWW behavior, receiving both values and producing a merged result
- `ring-topology-needs-multiple-sync-rounds` — A 3-node ring topology requires at least 2 `sync()` rounds to achieve full convergence, unlike a fully-connected topology which converges in 1
- `sync-is-idempotent` — Calling `sync()` multiple times on an already-converged cluster produces no new conflicts and preserves existing values
- `delete-propagates-as-tombstone` — `ReplicaNode.delete()` creates a tombstone that, once synced, causes `get()` to return `None` on all nodes in the cluster

