# File: raft-consensus/raft.py

**Date:** 2026-05-29
**Time:** 09:02

# `raft-consensus/raft.py` — Raft Consensus Algorithm Simulation

## Purpose

This file is a self-contained, single-threaded simulation of the Raft consensus algorithm as described in the Ongaro & Ousterhout paper (and Chapter 9 of DDIA). It owns the core protocol logic — leader election, log replication, and commit advancement — plus a `RaftCluster` harness that simulates message passing and network partitions without real networking. The simulation is tick-driven: time advances in discrete steps, making the behavior deterministic enough for testing.

## Key Components

### `LogEntry`

A simple value object holding `term`, `index`, and `command`. No behavior — it's just a record carried in the replicated log.

### `RaftNode`

The core state machine for a single Raft participant. Its state is organized into three categories that mirror the paper:

- **Persistent state** (`_current_term`, `_voted_for`, `_log`): Survives restarts in a real system. Here it just lives in memory, but the separation is intentional documentation.
- **Volatile state** (`_state`, `_commit_index`, `_last_applied`): Reconstructable from the log and peer communication.
- **Leader-only state** (`_next_index`, `_match_index`): Tracks replication progress per follower. Only meaningful when `_state == "leader"`.

Key methods and their contracts:

| Method | Role | Returns |
|--------|------|---------|
| `tick(elapsed_ms)` | Advances timers; fires elections or heartbeats | List of outbound messages (dicts) |
| `handle_request_vote(...)` | Processes a vote request from a candidate | `{term, vote_granted}` |
| `handle_append_entries(...)` | Processes a log replication / heartbeat RPC | `{term, success, match_index}` |
| `handle_vote_response(...)` | Candidate tallies a vote reply | None (side effects only) |
| `handle_append_response(...)` | Leader updates replication tracking | None (side effects only) |
| `client_request(command)` | Appends a command to the leader's log | `{success, entry, error}` |
| `get_committed_entries()` | Returns log entries at or below `_commit_index` | List of `LogEntry` (excludes sentinel) |

### `RaftCluster`

A synchronous test harness that wires nodes together. It owns message dispatch and simulates network partitions via `_partitioned`. Key methods:

- `tick(elapsed_ms)` — Collects messages from all non-partitioned nodes, delivers them synchronously (including responses within the same tick).
- `run_until_leader(max_ticks)` — Spins the clock until exactly one leader exists.
- `submit(command)` — Finds the leader and appends a client command.
- `run_until_committed(index, max_ticks)` — Spins until a majority have committed up to `index`.
- `partition_node` / `heal_node` — Toggle network isolation for a node.

## Patterns

**Tick-driven simulation.** Instead of real timers or async I/O, time advances via explicit `tick()` calls. This makes tests deterministic (modulo `random.randint` for election timeouts) and avoids threading complexity.

**Message-passing via dicts.** Outbound messages are plain dicts with a `type` field (`"request_vote"` or `"append_entries"`). The cluster dispatches based on `type` and delivers responses inline. This is a simplified version of the RPC layer a real implementation would have.

**State machine transitions.** The three states (`follower`, `candidate`, `leader`) are managed by `_become_follower`, `_become_candidate`, and `_become_leader`. Every transition resets the relevant timers and volatile state. The transitions follow the Raft paper exactly: follower → candidate (election timeout), candidate → leader (majority vote), any → follower (higher term seen).

**Sentinel log entry.** The log is initialized with a dummy entry at index 0, term 0. This eliminates boundary checks — `_last_log_index()` and `_last_log_term()` always have a valid entry to read, and `prev_log_index=0` always matches.

## Dependencies

- **Imports:** `random` only — used for randomized election timeouts in `_reset_election_timer`.
- **Imported by:** `test_raft.py` — the test suite exercises the cluster harness with elections, replication, and partition scenarios.

No external libraries. The entire protocol is self-contained.

## Flow

### Leader Election

1. A follower's `tick()` increments `_election_timer`. When it exceeds the randomized `_election_timeout`, the node calls `_become_candidate()` — increments term, votes for itself, and broadcasts `request_vote` messages.
2. Each recipient calls `handle_request_vote()`. It grants a vote if: (a) it hasn't voted this term (or already voted for this candidate), and (b) the candidate's log is at least as up-to-date (`_is_log_up_to_date`).
3. The cluster delivers the response back to the candidate via `handle_vote_response()`. If the candidate accumulates a strict majority, it calls `_become_leader()`.

### Log Replication

1. A client calls `client_request(command)` on the leader, which appends a `LogEntry` to its local log.
2. On the next heartbeat tick, the leader sends `append_entries` to each peer, computed by `_make_append_entries()`. The `prev_log_index` and `prev_log_term` fields allow the follower to verify log consistency.
3. The follower's `handle_append_entries()` checks consistency. On mismatch, it truncates its log and returns `success=False` with its current `match_index` so the leader can back up `next_index` and retry.
4. On success, the leader updates `_match_index` and calls `_advance_commit_index()`, which scans for the highest log index replicated to a majority **in the current term**.

### Commit Advancement

`_advance_commit_index` iterates from `_commit_index + 1` upward. For each index, it counts how many peers have `match_index >= n` (plus the leader itself). If a majority exists **and** the entry's term equals `_current_term`, `_commit_index` advances. The current-term check is the Raft safety property — leaders never commit entries from previous terms by counting replicas alone.

## Invariants

1. **Election safety.** A node grants at most one vote per term — enforced by `_voted_for` being checked in `handle_request_vote` and set atomically with the grant.
2. **Log matching.** If two logs contain an entry with the same index and term, all preceding entries are identical. Enforced by the `prev_log_index`/`prev_log_term` consistency check in `handle_append_entries` and truncation on mismatch.
3. **Leader completeness.** A candidate's log must be at least as up-to-date as the voter's log to win election — `_is_log_up_to_date` compares term first, then index.
4. **Current-term commit only.** `_advance_commit_index` skips entries where `self._log[n].term != self._current_term`, preventing the "commit from a previous term" safety violation (Raft paper §5.4.2).
5. **Step-down on higher term.** Every RPC handler checks `if term > self._current_term: self._become_follower(term)`. This is the universal "if you see a bigger term, you're stale" rule.
6. **Sentinel log entry.** Index 0 is always present and never removed, so `_last_log_index()` and `_last_log_term()` never fail on an empty log.

## Error Handling

Errors are returned as data, never raised as exceptions:

- `client_request` returns `{"success": False, "error": "not leader"}` if the node isn't the leader. There's no leader forwarding.
- `handle_append_entries` returns `{"success": False, "match_index": ...}` on log inconsistency. The `match_index` in the failure response tells the leader how far back to probe.
- `run_until_leader` and `run_until_committed` return `None` / `False` on timeout rather than raising.
- Network partitions silently drop messages — no error is surfaced. The protocol recovers via election timeouts and log catch-up once the partition heals.

There is no validation of inputs (e.g., no check that `candidate_term` is a positive integer). The code trusts its callers — appropriate for a simulation where the only caller is the cluster harness and the test suite.

## Topics to Explore

- [file] `raft-consensus/test_raft.py` — See how the cluster harness is exercised: election, replication under partitions, log convergence after healing
- [function] `raft-consensus/raft.py:_advance_commit_index` — The current-term-only commit rule is the subtlest safety property in Raft; trace through what happens if you remove the term check
- [general] `raft-leader-stickiness` — This implementation has no pre-vote or leader lease mechanism — explore how split-brain scenarios resolve and whether unnecessary elections occur
- [file] `two-phase-commit/two_phase_commit.py` — Compare the consensus approach: 2PC requires all participants, Raft only a majority — how does this affect partition tolerance?
- [general] `raft-log-compaction` — This implementation has no snapshotting or log truncation — consider how the log grows unboundedly and what a compaction extension would require

## Beliefs

- `raft-sentinel-log-entry` — The log is initialized with a sentinel `LogEntry(term=0, index=0)` at position 0 that is never removed, eliminating empty-log edge cases in all log access methods
- `raft-current-term-commit-only` — `_advance_commit_index` only commits entries whose term matches `_current_term`, implementing the Raft safety property from §5.4.2 that prevents committing prior-term entries by replica count alone
- `raft-single-tick-delivery` — `RaftCluster.tick()` collects outbound messages and delivers both the request and its response within the same tick invocation — there is no simulated network latency
- `raft-no-leader-forwarding` — `client_request` returns an error if the node is not the leader; there is no automatic forwarding to the current leader
- `raft-partition-via-set` — Network partitions are simulated by adding node IDs to `_partitioned`; partitioned nodes neither tick nor send/receive messages, modeling a complete network isolation

