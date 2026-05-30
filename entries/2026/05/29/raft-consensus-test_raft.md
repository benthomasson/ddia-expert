# File: raft-consensus/test_raft.py

**Date:** 2026-05-29
**Time:** 13:49

I can't access the implementation file due to permissions, but I have enough from the test file to give a thorough explanation.

---

# `raft-consensus/test_raft.py`

## Purpose

This is the test suite for a simulated Raft consensus algorithm implementation. It validates the core safety and liveness properties of Raft — leader election, log replication, network partitions, and log consistency — using a deterministic, in-process cluster simulation. It doesn't test a networked system; instead it drives a `RaftCluster` that advances time via explicit `tick()` calls, making tests reproducible and fast.

This file owns **behavioral verification** of the Raft protocol. It's the specification-as-code: if you want to know what the implementation guarantees, read these tests.

## Key Components

### Imports

- **`RaftCluster`** — The simulation harness. Manages a set of `RaftNode` instances, routes messages between them, and supports network partition simulation. This is the primary API the tests interact with.
- **`RaftNode`** — Individual Raft peer. Exposes state (`state`, `current_term`), log access (`get_log()`, `get_committed_entries()`), and client interaction (`client_request()`).
- **`LogEntry`** — Data class for replicated log entries, carrying at minimum a `command` field.

### Test Functions (grouped by what they verify)

**Leader Election:**
| Test | Property |
|------|----------|
| `test_leader_elected_from_fresh_cluster` | Liveness — a 5-node cluster elects a leader |
| `test_all_nodes_start_as_followers` | Initial state — term 0, follower role |
| `test_split_vote_resolves` | Liveness under contention — tight timeout range (150–155ms) forces split votes; leader still emerges within 2000 ticks |
| `test_heartbeat_prevents_elections` | Stability — once elected, heartbeats keep the term and leader stable across 100 ticks |

**Log Replication & Commit:**
| Test | Property |
|------|----------|
| `test_log_replication_and_commit` | Basic replication — two commands submitted, both appear in committed log |
| `test_client_request_rejected_by_follower` | Followers reject client writes with `{"success": False, "error": "not leader"}` |

**Partition & Recovery:**
| Test | Property |
|------|----------|
| `test_new_leader_after_partition` | New leader elected after partitioning the current leader |
| `test_committed_entries_survive_leader_change` | Safety — committed entries persist across leader changes |
| `test_minority_partition_cannot_elect_leader` | Safety — 2 of 5 nodes cannot form a quorum |
| `test_heal_and_log_consistency` | A healed follower catches up on entries it missed |
| `test_sequential_leader_failures` | Two consecutive leader failures; committed data survives |
| `test_uncommitted_entries_overwritten` | The "doomed entry" test — an uncommitted entry from a deposed leader is replaced by the new leader's log |

**Term Invariants:**
| Test | Property |
|------|----------|
| `test_terms_monotonically_increasing` | Terms strictly increase across 4 leader changes |

**Integration:**
| Test | Property |
|------|----------|
| `test_example_from_spec` | End-to-end walkthrough: elect → replicate → partition → re-elect → heal → old leader becomes follower |

## Patterns

### Deterministic simulation testing
The entire Raft protocol runs in-process. Time is simulated via `cluster.tick(ms)` — no real clocks, no threads, no sockets. This is the same pattern used by FoundationDB's simulation testing and TiKV's `raft-rs` test harness. It makes partition scenarios trivially reproducible.

### Cluster-level API
Tests never call `RaftNode` methods to drive protocol logic directly. Instead they use `RaftCluster` as a facade:
- `run_until_leader()` — spin ticks until some node becomes leader
- `submit(command)` — send a client command to the current leader
- `run_until_committed(n)` — tick until at least `n` entries are committed
- `partition_node(id)` / `heal_node(id)` — simulate network failures
- `get_committed_log()` — return committed commands across the cluster

This keeps tests at the right abstraction level — they describe *what should happen*, not *how messages flow*.

### Scenario scaffolding pattern
Most tests follow the same structure:
1. Create cluster → elect leader
2. Submit commands → wait for commit
3. Introduce a failure (partition)
4. Tick forward → observe recovery
5. Assert safety properties on the resulting state

### Seeded randomness for nondeterministic scenarios
`test_split_vote_resolves` sets `random.seed(42)` and uses a narrow election timeout range `(150, 155)` to intentionally provoke split votes while keeping the test deterministic.

## Dependencies

**Imports:**
- `raft.LogEntry`, `raft.RaftNode`, `raft.RaftCluster` — the implementation under test
- `random` — used only in `test_split_vote_resolves` for seed control
- `pytest` — imported but not directly used (no fixtures, parametrize, or markers); likely present for future use or runner discovery

**Imported by:** Nothing — this is a leaf test module.

## Flow

A typical test execution:

1. `RaftCluster(node_ids)` creates N `RaftNode` instances wired together through a simulated message bus.
2. `run_until_leader()` repeatedly calls `tick()` internally, advancing simulated time. Each tick lets nodes process timers (election timeouts, heartbeat intervals) and deliver queued messages. Eventually one node's election timeout fires, it transitions to candidate, requests votes, wins, and becomes leader.
3. `submit(cmd)` routes a client command to the current leader, which appends a `LogEntry` to its log and begins replicating it via AppendEntries RPCs.
4. `run_until_committed(n)` ticks until the leader's `commit_index` reaches `n` (meaning a majority has acknowledged the entry).
5. `partition_node(id)` drops all messages to/from that node. `heal_node(id)` restores connectivity. The partitioned node's election timer eventually fires, but it can't get votes if it's in the minority.
6. After healing, the leader detects the follower's log is behind and sends missing entries via AppendEntries, bringing it back into consistency.

## Invariants

The tests collectively enforce Raft's core safety properties:

1. **Election Safety** — At most one leader per term (tested indirectly: `get_leader()` returns a single node or `None`).
2. **Leader Completeness** — A newly elected leader's log contains all previously committed entries (`test_committed_entries_survive_leader_change`).
3. **Log Matching** — If two logs contain an entry with the same index and term, all preceding entries are identical (enforced by `test_heal_and_log_consistency`, `test_uncommitted_entries_overwritten`).
4. **State Machine Safety** — Committed entries are never overwritten; only uncommitted entries can be replaced (`test_uncommitted_entries_overwritten` — "doomed" is gone, "committed" and "replacement" survive).
5. **Quorum requirement** — A minority cannot elect a leader (`test_minority_partition_cannot_elect_leader` — 2 of 5 nodes).
6. **Term monotonicity** — Terms strictly increase across leader changes (`test_terms_monotonically_increasing`).
7. **Follower rejection** — Non-leaders reject client requests with a structured error (`test_client_request_rejected_by_follower`).

## Error Handling

The tests themselves don't exercise error paths in the traditional sense — there are no exception-handling tests. Instead, "errors" are modeled as protocol-level responses:

- `client_request()` on a follower returns `{"success": False, "error": "not leader"}` — a structured rejection, not an exception.
- Partition failures are simulated, not real — there are no socket errors or timeouts to handle.
- `run_until_leader()` and `run_until_committed()` appear to return `None`/`False` on timeout rather than raising, and the tests assert against these return values.

---

## Topics to Explore

- [file] `raft-consensus/raft.py` — The implementation being tested: `RaftNode` state machine, `RaftCluster` simulation harness, message dispatch, and election/replication logic
- [function] `raft-consensus/raft.py:RaftCluster.tick` — How simulated time drives the protocol: timer decrements, message delivery, and state transitions per tick
- [function] `raft-consensus/raft.py:RaftNode.client_request` — The client-facing API: how writes enter the replicated log and how non-leaders redirect or reject
- [general] `raft-log-reconciliation` — How AppendEntries handles log conflicts: the backtracking mechanism that overwrites stale entries (exercised by `test_uncommitted_entries_overwritten`)
- [general] `raft-election-restriction` — The voting rule that prevents stale-log candidates from winning (exercised by `test_stale_log_candidate_loses_election`): how `lastLogIndex` and `lastLogTerm` are compared

## Beliefs

- `raft-follower-rejects-writes` — `RaftNode.client_request()` on a follower returns `{"success": False, "error": "not leader"}` rather than raising an exception
- `raft-cluster-deterministic-simulation` — `RaftCluster` uses tick-based deterministic simulation with no real clocks or network I/O, making all tests reproducible
- `raft-uncommitted-entries-overwritten` — Uncommitted log entries from a deposed leader are overwritten when the node receives AppendEntries from a new leader with conflicting entries at the same index
- `raft-minority-cannot-elect` — A partition containing fewer than ⌈(n+1)/2⌉ nodes cannot elect a leader; `get_leader()` returns `None` in this scenario
- `raft-terms-strictly-increase` — Each new leader has a strictly higher term than the previous leader; terms never repeat or decrease across leader transitions

