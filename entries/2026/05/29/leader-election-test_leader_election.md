# File: leader-election/test_leader_election.py

**Date:** 2026-05-29
**Time:** 13:19

## Purpose

This is the **test suite for a Bully Algorithm leader election implementation**. It validates that the `BullyElectionCluster` correctly implements the core properties of the Bully Algorithm: the highest-ID alive node always becomes leader, leadership transfers on failure, and a recovered high-ID node "bullies" its way back to leadership.

The file serves as both a pytest-compatible test module and a standalone runner (via `__main__`), making it usable in CI and ad-hoc debugging alike.

## Key Components

### Test Functions

| Test | What it verifies |
|------|-----------------|
| `test_initial_election_highest_wins` | Node 5 wins initial election in a 5-node cluster |
| `test_all_nodes_agree_on_leader` | All nodes converge — followers point to the same leader, leader reports `"leader"` state |
| `test_leader_failure_triggers_new_election` | Failing node 5 causes node 4 to take over |
| `test_recovered_node_takes_over` | Recovering node 5 reclaims leadership (the "bully" property) |
| `test_multiple_failures` | Failing nodes 5 and 4 leaves node 3 as leader |
| `test_single_survivor` | Only node 1 alive → node 1 is leader |
| `test_single_node_cluster` | Degenerate cluster of size 1 handles election correctly |
| `test_election_history` | `get_election_history()` records at least 3 leadership transitions across fail/recover cycles |
| `test_terms_increase_monotonically` | Election terms never decrease across successive history entries |
| `test_example_from_spec` | End-to-end scenario exercising initial election → failure → recovery → multi-failure |

### Standalone Runner (lines 107–124)

A manual test harness that runs all tests sequentially, prints PASS/FAIL per test, and exits with code 1 on any failure. This mirrors what pytest does but allows running without pytest installed.

## Patterns

**Deterministic simulation.** The cluster is not running real timers or threads. Instead, `run_until_leader(start_time=N)` advances a simulated clock, making tests fully deterministic and free of flaky timing issues. Each test controls the timeline explicitly.

**Fail-then-assert.** Tests follow a consistent structure: set up a cluster → trigger a failure/recovery → run election → assert the expected leader. This isolates each Bully Algorithm property into a single test.

**State-based verification.** Rather than inspecting messages, most tests check the resulting state (`get_cluster_state()`, `get_election_history()`) — treating the cluster as a black box and verifying its externally visible properties.

**Cumulative scenario.** `test_example_from_spec` is a full integration test that chains multiple operations (elect → fail → re-elect → recover → re-elect → multi-fail → re-elect) in sequence, verifying the complete lifecycle.

## Dependencies

**Imports:**
- `sys` — only for `sys.exit(1)` in the standalone runner
- `leader_election.Message`, `leader_election.BullyNode`, `leader_election.BullyElectionCluster` — the implementation under test. Only `BullyElectionCluster` is actually used; `Message` and `BullyNode` are imported but unused in the tests (they're exercised indirectly through the cluster)

**Imported by:** Nothing — this is a leaf test file.

## Flow

Each test follows this pattern:

1. **Construct** a `BullyElectionCluster` with a list of node IDs and timing parameters (`heartbeat_interval=3`, `election_timeout=10`).
2. **Run** `cluster.run_until_leader()` to simulate the election process until a leader emerges.
3. Optionally **mutate** the cluster via `fail_node(id)` or `recover_node(id)`.
4. **Re-run** `run_until_leader(start_time=N)` with a later simulated time to trigger re-election.
5. **Assert** the returned leader ID matches the expected highest-alive node.
6. Optionally **inspect** cluster-wide state or history for deeper invariant checks.

The `start_time` parameter is critical — it provides temporal separation between election rounds so that heartbeat timeouts from the previous leader have expired and surviving nodes detect the failure.

## Invariants

1. **Bully invariant**: The leader is always the highest-ID alive node. Every test implicitly validates this.
2. **Consensus**: After `run_until_leader()`, all alive nodes agree on the leader (`test_all_nodes_agree_on_leader`).
3. **Monotonic terms**: Election terms never decrease across the history (`test_terms_increase_monotonically`).
4. **History completeness**: At least 3 history entries are recorded across 3 election rounds (`test_election_history`).
5. **Leader/follower role separation**: The leader node reports state `"leader"`, all others report `"follower"`.

## Error Handling

Tests use bare `assert` with descriptive f-string messages — no try/except within individual tests. The standalone runner catches all exceptions per-test to continue running the full suite and report aggregate results. There's no retry logic or tolerance for flakiness, which is appropriate given the deterministic simulation model.

## Topics to Explore

- [file] `leader-election/leader_election.py` — The implementation of `BullyElectionCluster`, `BullyNode`, and the simulated message-passing that makes these deterministic tests possible
- [function] `leader-election/leader_election.py:run_until_leader` — How the simulated clock advances and when election termination is detected
- [function] `leader-election/leader_election.py:fail_node` — How node failure is modeled (partition vs crash, message handling for failed nodes)
- [general] `bully-algorithm-vs-raft` — Compare the Bully Algorithm's simplicity (highest ID wins) with Raft's log-based leader election in `raft-consensus/`
- [file] `leader-election/test_smoke.py` — Likely a lighter smoke test; worth comparing coverage with this full suite

## Beliefs

- `bully-election-highest-id-wins` — The Bully Algorithm implementation guarantees the highest-ID alive node becomes leader, validated across initial election, failure recovery, and multi-failure scenarios
- `bully-tests-use-deterministic-simulation` — Tests control time via `start_time` parameter rather than real timers, making the election test suite fully deterministic with no sleep-based timing
- `bully-election-terms-monotonic` — Election terms in the history are enforced to be monotonically non-decreasing across successive elections
- `bully-test-imports-unused-symbols` — `Message` and `BullyNode` are imported but never directly referenced in any test function; all tests interact through `BullyElectionCluster`

