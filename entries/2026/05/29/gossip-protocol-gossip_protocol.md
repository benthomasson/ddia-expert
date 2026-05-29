# File: gossip-protocol/gossip_protocol.py

**Date:** 2026-05-29
**Time:** 08:55

# `gossip-protocol/gossip_protocol.py`

## Purpose

This file implements a **gossip-based failure detection and membership protocol**, the kind described in DDIA Chapter 8 (distributed systems) and used by systems like Cassandra and Riak. It owns two responsibilities: (1) maintaining a consistent cluster membership view across nodes via epidemic-style information dissemination, and (2) detecting node failures through heartbeat timeouts with a three-state lifecycle (`alive → suspected → dead → removed`).

## Key Components

### `GossipNode`

A single participant in the gossip cluster. Each node maintains its own local copy of the full membership list — there is no shared state.

**Constructor** accepts three timeout thresholds that govern the failure detection state machine:
- `t_suspect` — time without a heartbeat update before marking a node `suspected`
- `t_dead` — time before escalating to `dead`
- `t_cleanup` — time before removing a dead node's entry entirely

**Core methods:**

| Method | Contract |
|--------|----------|
| `join(seed_node)` | Bootstraps membership by copying the seed's list and registering self with the seed. Mutates both nodes. |
| `leave()` | Voluntary departure — marks self `dead` and sets `_leaving` flag so the cluster can broadcast the death before deactivation. |
| `heartbeat(current_time)` | Increments own counter and updates timestamp. No-ops if already dead. |
| `send_gossip()` | Returns a deep copy of the membership list (safe to hand to another node). |
| `receive_gossip(membership_list, current_time)` | Merge logic — the heart of the protocol. |
| `detect_failures(current_time)` | Applies timeout-based state transitions and garbage-collects old dead entries. Returns a dict of status changes. |

### `GossipCluster`

A simulation harness that orchestrates multiple `GossipNode` instances. Not a distributed runtime — it's a deterministic, single-process simulator with a seeded RNG for reproducible gossip partner selection.

**Key methods:**

| Method | Contract |
|--------|----------|
| `add_node(node_id)` | Creates a node and joins it to any existing alive node. Returns the node. |
| `remove_node(node_id)` | Simulates a crash — no leave message, node just stops heartbeating. |
| `gossip_round(current_time)` | One tick: handle leavers, heartbeat all active nodes, random pairwise exchange, then failure detection. |
| `run_rounds(num_rounds, start_time)` | Runs multiple rounds, collecting per-node membership snapshots at each tick. |

## Patterns

**Crux of the protocol — `receive_gossip` merge rule.** The merge uses **heartbeat counter monotonicity** as the conflict resolution mechanism. A remote entry is accepted only when its heartbeat counter exceeds the local counter, which prevents stale information from overwriting fresh state. There's a special case: a `dead` status with an *equal* counter is also accepted, ensuring voluntary leave notifications propagate even without a heartbeat bump.

**Deep-copy isolation.** Every boundary crossing (`send_gossip`, `join`, `get_membership_list`) uses `copy.deepcopy` to prevent aliased mutation between nodes. This is critical for simulation correctness — without it, all nodes would share the same dict objects.

**Two-phase voluntary leave.** `leave()` doesn't immediately deactivate the node. It sets `_leaving = True`, and `gossip_round` broadcasts the leave message to all peers *before* setting `_alive = False`. This ensures at least one round of propagation.

**Deterministic simulation.** `GossipCluster` takes an optional `seed` for `random.Random`, making test scenarios reproducible despite the random peer selection.

## Dependencies

**Imports:** Only stdlib — `copy` for isolation and `random` for peer selection. No external dependencies.

**Imported by:** Test files (`test_gossip_protocol.py`, `tester_test_gossip_protocol.py`) exercise both `GossipNode` directly and `GossipCluster` for integration scenarios.

## Flow

A typical simulation round (`gossip_round`) proceeds in four phases:

1. **Leave broadcast** — Any node with `_leaving=True` sends its membership (containing its own `dead` status) to every active peer, then deactivates.
2. **Heartbeat** — Each active node increments its own counter.
3. **Pairwise exchange** — Each active node picks one random peer. They exchange full membership lists bidirectionally. This is the epidemic spread mechanism — information propagates exponentially across the cluster.
4. **Failure detection** — Every active node scans its membership list and applies timeout-based transitions: `alive → suspected` (after `t_suspect`), `suspected → dead` (after `t_dead`), `dead → removed` (after `t_cleanup`).

Information convergence relies on the random pairwise exchanges accumulating across rounds. After O(log N) rounds, all nodes will typically have consistent views.

## Invariants

- **Heartbeat counter is monotonically increasing** for any given node — it never decreases. The merge rule relies on this.
- **Status transitions are one-directional**: `alive → suspected → dead → removed`. A node can skip `suspected` (direct to `dead` via timeout or gossip), but never goes backward — with one exception: a remote gossip with a higher counter and `alive` status will reset a `suspected` node back to `alive` (the node recovered).
- **Dead nodes don't heartbeat.** `heartbeat()` exits early if status is `dead`.
- **New nodes with `dead` status are not added.** `receive_gossip` skips unknown nodes that arrive already dead — prevents zombie entries.
- **Self-entry is never failure-checked.** `detect_failures` skips `nid == self.node_id`.
- **Leaving nodes broadcast to all peers**, not just a random one, ensuring deterministic propagation of voluntary departures.

## Error Handling

There is essentially none — this is a simulation, not production code. No exceptions are raised or caught. Invalid inputs (e.g., joining a cluster with a crashed seed node, adding a duplicate node ID) would produce incorrect state silently rather than failing loudly. The code trusts that `GossipCluster` drives the simulation correctly.

## Topics to Explore

- [file] `gossip-protocol/test_gossip_protocol.py` — See how failure detection convergence, voluntary leave, and network partitions are tested
- [function] `gossip-protocol/gossip_protocol.py:receive_gossip` — The merge rule is the most subtle part; trace through scenarios where counter equality matters for death propagation
- [general] `phi-accrual-failure-detector` — How production systems (Cassandra) replace the fixed-threshold approach here with a probabilistic one that adapts to network conditions
- [file] `conflict-free-replicated-data-types/crdts.py` — CRDTs solve a related problem (convergent merge without coordination); compare the merge semantics
- [general] `swim-protocol` — SWIM is the modern evolution of basic gossip failure detection, adding suspicion-based protocol extensions and piggyback dissemination

## Beliefs

- `gossip-merge-uses-heartbeat-monotonicity` — `receive_gossip` only accepts remote state when the remote heartbeat counter strictly exceeds the local counter, except for death notifications at equal counters
- `gossip-node-status-lifecycle` — Node status follows `alive → suspected → dead → removed` with timeouts `t_suspect < t_dead < t_cleanup` governing transitions
- `gossip-deep-copy-isolation` — All inter-node data transfer uses `copy.deepcopy` to prevent shared mutable state between simulated nodes
- `gossip-voluntary-leave-broadcasts-to-all` — A leaving node sends its death status to every active peer (not just a random one) before deactivating, unlike normal gossip which uses random pairwise exchange
- `gossip-dead-nodes-filtered-on-receive` — When receiving gossip, unknown nodes with `dead` status are silently dropped to prevent zombie membership entries

