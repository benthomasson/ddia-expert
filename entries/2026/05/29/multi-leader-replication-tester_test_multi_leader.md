# File: multi-leader-replication/tester_test_multi_leader.py

**Date:** 2026-05-29
**Time:** 13:54

## Purpose

This is the **standalone test harness** for the multi-leader replication module. It validates that the `multi_leader.py` implementation correctly handles the core challenges of multi-leader replication as described in DDIA: conflict resolution, topology-aware propagation, convergence, and idempotent sync.

The `tester_` prefix distinguishes it from `test_multi_leader.py` (likely a pytest-compatible suite). This file is self-contained — it runs directly via `python tester_test_multi_leader.py` with a manual test runner at the bottom, producing pass/fail output and setting the exit code.

## Key Components

### Test Functions

| Test | What it validates |
|------|-------------------|
| `test_basic_replication_no_conflict` | Non-conflicting writes across 3 nodes converge after one sync, with zero conflict records |
| `test_lww_conflict_higher_timestamp_wins` | LWW resolves concurrent writes by choosing the higher Lamport timestamp |
| `test_lww_tiebreak_by_node_id` | When timestamps tie, the lexicographically higher `node_id` wins — deterministic tiebreak |
| `test_custom_merge` | A user-supplied merge function (counter addition) replaces LWW for domain-specific resolution |
| `test_tombstone_delete` | Deletes propagate as tombstones; `get()` returns `None` after sync |
| `test_ring_topology` | Ring topology requires `>=2` sync rounds (hop-by-hop) vs. all-to-all |
| `test_conflict_logging` | `ConflictRecord` captures key, both values, and the strategy used |
| `test_lamport_clock_ordering` | Local writes produce monotonically increasing timestamps; remote apply advances the clock past the remote timestamp |
| `test_idempotency` | Double-syncing is a no-op — no duplicate entries or spurious conflicts |
| `test_convergence_many_keys` | 5 nodes × 20 keys each all converge after sync |
| `test_custom_merge_repeated_sync_converges` | Custom merge doesn't "explode" on repeated syncs (e.g., counter doesn't keep doubling) |

### Manual Test Runner (lines 147–163)

A `__main__` block that iterates over all test functions, catches exceptions to report `PASS`/`FAIL`, and exits with code 1 on any failure. This pattern appears across all `tester_*.py` files in the repo — it's the project's convention for CI-friendly test execution without pytest.

## Patterns

**Cluster-as-test-fixture**: Every test instantiates a `MultiLeaderCluster` with a list of node IDs. The cluster manages node lifecycle, topology wiring, and sync orchestration. Tests never manually wire nodes together.

**Sync-then-assert**: The dominant pattern is write → `cluster.sync()` → assert convergence. `sync()` is a discrete step, not background — tests control exactly when replication happens.

**Convergence oracle**: `cluster.all_converged()` provides a single boolean check that all nodes agree on all keys. Tests use this as the primary correctness assertion alongside spot-checks on specific keys.

**Strategy injection**: Conflict resolution strategy is passed at cluster construction time (`ConflictStrategy.CUSTOM_MERGE` + `merge_fn`), not per-operation. This mirrors DDIA's model where conflict strategy is a system-level policy.

## Dependencies

**Imports from `multi_leader`:**
- `ReplicaNode` — individual node, used directly in `test_lamport_clock_ordering`
- `MultiLeaderCluster` — the cluster orchestrator, used in every other test
- `ConflictStrategy` — enum for LWW vs. custom merge
- `Topology` — enum for ALL_TO_ALL (default) vs. RING
- `ConflictRecord` — dataclass for conflict audit trail

**Imported by:** Nothing — this is a leaf test file.

## Flow

1. Each test function constructs a cluster with specific parameters (node count, topology, strategy).
2. Writes happen via `cluster.node(id).put(key, value)`, which records the change locally with a Lamport timestamp.
3. `cluster.sync()` triggers one round of replication across the configured topology. For ring topology, `cluster.sync_until_converged()` loops sync rounds until convergence.
4. Assertions check both individual key values and global convergence.
5. The `__main__` runner collects results and produces a summary line like `11 passed, 0 failed`.

## Invariants

- **LWW determinism**: Timestamp comparison is `(timestamp, node_id)` — strictly deterministic, no randomness. `test_lww_tiebreak_by_node_id` encodes this: `"b" > "a"` lexicographically, so node b wins ties.
- **Lamport clock monotonicity**: `test_lamport_clock_ordering` asserts `ts1 < ts2 < ts3` for sequential local writes, and that the clock advances past any remote timestamp seen.
- **Idempotent sync**: Syncing twice produces the same state as syncing once. No duplicate conflicts, no value drift.
- **Custom merge stability**: After initial convergence, additional sync rounds must not change values. This prevents the classic bug where a commutative merge (like addition) gets re-applied on every sync cycle.
- **Ring topology requires multi-hop**: The test asserts `rounds >= 2`, encoding the invariant that ring topology can't converge in a single round for 3+ nodes.

## Error Handling

Minimal — this is a test file. Each test function either returns (pass) or raises `AssertionError` (fail). The `__main__` runner catches all exceptions via a bare `except Exception`, prints the failure, and continues to the next test. Exit code 1 signals any failure to the caller.

There's no retry logic, no cleanup, and no teardown — clusters are ephemeral in-memory objects that get garbage collected.

## Topics to Explore

- [file] `multi-leader-replication/multi_leader.py` — The implementation under test: how `sync()`, conflict resolution, and topology routing actually work
- [function] `multi-leader-replication/multi_leader.py:MultiLeaderCluster.sync` — The sync mechanism that drives replication rounds and applies changes across nodes
- [function] `multi-leader-replication/multi_leader.py:ReplicaNode.apply_remote_change` — Where conflict detection and resolution actually happen — the core of multi-leader complexity
- [general] `topology-propagation-model` — How ALL_TO_ALL vs. RING topologies affect convergence speed and message complexity (DDIA Chapter 5)
- [file] `multi-leader-replication/test_multi_leader.py` — The pytest-compatible counterpart; compare what it covers vs. this standalone harness

## Beliefs

- `multi-leader-lww-tiebreak-uses-node-id` — LWW conflict resolution uses `(timestamp, node_id)` tuple comparison, with lexicographically higher node_id winning ties
- `multi-leader-custom-merge-must-be-idempotent-across-syncs` — Custom merge functions are applied once per conflict; repeated sync rounds must not re-trigger the merge on already-converged values
- `multi-leader-ring-topology-needs-multiple-rounds` — Ring topology requires at least 2 sync rounds to propagate a write across 3 nodes, unlike all-to-all which converges in 1
- `multi-leader-sync-is-discrete-not-continuous` — Replication happens only when `sync()` is explicitly called; there is no background replication thread
- `multi-leader-conflict-record-captures-both-values` — `ConflictRecord` stores both `local_value` and `remote_value` along with the key and resolution strategy, providing a full audit trail

