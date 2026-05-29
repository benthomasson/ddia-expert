# File: leader-election/leader_election.py

**Date:** 2026-05-29
**Time:** 08:58

# `leader-election/leader_election.py`

## Purpose

This file implements the **Bully Algorithm** for leader election in distributed systems — a classic protocol from DDIA Chapter 8 (The Trouble with Distributed Systems) and Chapter 9 (Consistency and Consensus). It provides both the node-level protocol logic (`BullyNode`) and a deterministic simulation harness (`BullyElectionCluster`) that drives the protocol in a single-threaded, tick-based loop. The simulation avoids real networking and timers entirely, making the algorithm testable and observable.

## Key Components

### `Message`

A plain data object representing a protocol message. Four message types exist:

| Type | Meaning |
|------|---------|
| `ELECTION` | "I'm starting an election" — sent to all higher-ID nodes |
| `ALIVE` | "I'm here, back off" — response from a higher-ID node to an ELECTION |
| `COORDINATOR` | "I won, I'm the leader" — broadcast by the winner to all nodes |
| `HEARTBEAT` | "I'm still alive as leader" — periodic liveness signal |

Messages carry `sender_id`, `receiver_id`, `term` (monotonic election counter), and `timestamp` (logical tick).

### `BullyNode`

The core protocol state machine. Each node is in one of three states: `"follower"`, `"candidate"`, or `"leader"`.

**Key methods:**

- **`receive_message(msg) -> list[Message]`** — The main dispatch. Handles all four message types and returns any response messages. This is a pure function of current state + incoming message, making it easy to reason about.

- **`tick(current_time) -> list[Message]`** — Timer-driven transitions. Leaders emit heartbeats on interval. Followers start elections when heartbeat timeout expires. Candidates either declare victory (no ALIVE received) or step down (ALIVE received) after half the election timeout.

- **`start_election(current_time) -> list[Message]`** — Increments term, transitions to candidate, sends ELECTION to all higher-ID nodes. If no higher nodes exist, declares victory immediately — this is the key Bully optimization.

- **`declare_victory(current_time) -> list[Message]`** — Transitions to leader, broadcasts COORDINATOR to all other nodes.

- **`set_available(available)`** — Simulates crash/recovery. When set to `False`, the node drops to follower and stops processing.

### `BullyElectionCluster`

The simulation harness. Owns a dict of `BullyNode` instances keyed by node ID, a logical clock, and election history.

**Key methods:**

- **`tick(current_time)`** — The simulation loop. Collects tick-generated messages from all nodes, then enters a delivery loop: deliver messages, collect responses, repeat until quiescent. After delivery, runs split-brain resolution.

- **`_resolve_split_brain(current_time)`** — Safety net: if multiple nodes claim leadership, lower-ID leaders are forced to restart elections. This handles edge cases the basic protocol can miss in a synchronous simulation.

- **`run_until_leader(start_time, max_ticks) -> int | None`** — Convenience method that ticks until all available nodes agree on a single leader, or returns `None` after `max_ticks`.

- **`fail_node(node_id)` / `recover_node(node_id)`** — Simulate crashes and recoveries. Recovery immediately triggers an election, which is the Bully Algorithm's defining behavior: a recovered high-ID node will preempt any current leader.

## Patterns

**Tick-based discrete simulation.** No threads, no async, no real timers. Time advances via explicit `tick(t)` calls, making the protocol fully deterministic and reproducible. This is the standard pattern across this repo's distributed systems implementations.

**Message-passing state machine.** Nodes communicate exclusively through `Message` objects returned from `receive_message` and `tick`. There's no shared mutable state between nodes — the cluster harness is the message bus.

**Eager delivery loop.** `BullyElectionCluster.tick()` delivers messages in a `while all_messages` loop, processing all cascading responses within a single tick. This models synchronous message delivery — messages sent at time `t` are received at time `t`.

**Split-brain as post-hoc correction.** Rather than trying to prevent split-brain through the protocol alone, the cluster detects it after each tick and resolves it by forcing lower-ID leaders to re-elect. This is a pragmatic simulation choice — real systems would rely on the protocol's properties plus network timeouts.

## Dependencies

**Imports:** None — this is a zero-dependency, pure-Python implementation.

**Imported by:** `leader-election/test_leader_election.py` — the test suite exercises election, failure, recovery, and split-brain scenarios.

## Flow

A typical election proceeds as:

1. A follower's heartbeat timer expires → `tick()` calls `start_election()`.
2. `start_election` sends `ELECTION` messages to all nodes with higher IDs.
3. If a higher-ID node receives `ELECTION`, it responds with `ALIVE` and starts its own election.
4. The original candidate sees `ALIVE`, sets `_got_alive = True`, and will step down on next tick.
5. The highest available node finds no higher nodes (or no `ALIVE` responses arrive) and calls `declare_victory`.
6. `declare_victory` broadcasts `COORDINATOR` to all nodes, which accept the new leader and update `_leader_id`.
7. The leader begins sending `HEARTBEAT` messages every `heartbeat_interval` ticks.

**Recovery flow:** When `recover_node` is called, the node immediately starts an election. If it has the highest ID, it will preempt the current leader — this is the "bully" behavior.

## Invariants

- **Highest-ID-wins:** The Bully Algorithm guarantees the highest available node becomes leader. `start_election` only sends to higher-ID nodes; `declare_victory` only happens when no higher node responded.
- **Term monotonicity:** `_current_term` is incremented on every `start_election` call and never decremented. Followers adopt the leader's term from COORDINATOR messages.
- **Unavailable nodes are inert:** `receive_message` and `tick` both early-return `[]` when `_available` is `False`. The cluster harness also checks `is_available()` before delivering messages.
- **COORDINATOR rejection:** A node rejects a COORDINATOR from a lower-ID sender by starting its own election (line ~87). This prevents a stale low-ID node from claiming leadership.
- **Election timeout split:** Candidates wait `election_timeout // 2` for ALIVE responses before deciding. This is shorter than the follower's full `election_timeout` to avoid cascading delays.

## Error Handling

There is essentially none — and that's intentional. This is a simulation, not production code. Invalid node IDs in messages are silently dropped (`self.nodes.get(msg.receiver_id)` returns `None`). No exceptions are raised. The design assumes the harness is the only caller and will provide valid inputs.

## Topics to Explore

- [file] `leader-election/test_leader_election.py` — How the test suite validates election convergence, failure recovery, and split-brain resolution
- [file] `raft-consensus/raft.py` — Compare Bully's "highest ID wins" approach with Raft's voting-based consensus; Raft is the more practical algorithm covered later in DDIA
- [function] `leader-election/leader_election.py:_resolve_split_brain` — Understand when the basic Bully protocol can produce split-brain in synchronous simulation and why the post-hoc fix is needed
- [general] `bully-vs-ring-election` — The Bully Algorithm trades message complexity (O(n²) worst case) for simplicity; compare with ring-based election algorithms
- [file] `fencing-tokens/fencing_tokens.py` — DDIA's recommendation for making leader election safe: even with a leader, fencing tokens prevent stale leaders from corrupting state

## Beliefs

- `bully-highest-id-wins` — The Bully Algorithm always elects the highest available node ID as leader; no voting or quorum is involved
- `bully-synchronous-delivery` — All messages generated within a single `tick()` call are delivered and fully resolved (including cascading responses) before the tick returns
- `bully-no-external-dependencies` — The implementation is pure Python with zero imports; all networking and timing are simulated through the tick-based harness
- `bully-split-brain-resolved-post-hoc` — Split-brain is detected and resolved by the cluster harness after each tick, not prevented by the node-level protocol alone
- `bully-recovery-triggers-election` — When a failed node recovers via `recover_node`, it immediately starts an election, potentially preempting the current leader

