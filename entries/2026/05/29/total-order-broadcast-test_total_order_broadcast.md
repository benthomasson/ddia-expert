# File: total-order-broadcast/test_total_order_broadcast.py

**Date:** 2026-05-29
**Time:** 14:23

I can't read the implementation file due to permissions, but I have enough from the test file and its imports to give a thorough explanation.

---

## Purpose

`test_total_order_broadcast.py` is the test suite for a **total-order broadcast (TOB) protocol** implementation — one of the 37 DDIA reference modules. It validates that the system delivers messages to all non-faulty nodes in the **same order**, which is the defining property of total-order broadcast (also called atomic broadcast).

The file exercises the protocol across four layers: the low-level **consensus instance** (single-slot Paxos), the **TOBNode/TOBCluster** network simulation, **fault tolerance** (crash/recovery), and a **linearizable register** built on top of TOB as a replicated state machine.

## Key Components

The test file imports four classes and exercises each:

| Class | Role in tests |
|---|---|
| `ConsensusInstance` | Single Paxos slot — tested directly in `test_consensus_competing_proposers` for prepare/accept semantics |
| `TOBNode` | Individual node — accessed via `cluster.get_node(i)` to register delivery callbacks |
| `TOBCluster` | Simulated network of TOBNodes — the primary test fixture, provides `broadcast()`, `run_until_delivered()`, `crash_node()`, `recover_node()`, `verify_total_order()` |
| `LinearizableRegister` | Key-value register built on TOB — wraps the cluster to provide `write()`, `read()`, `compare_and_set()` |

There are no test fixtures or shared setup — each test constructs a fresh `TOBCluster(3)`, keeping tests fully independent.

## Patterns

**Layered testing from consensus primitives up to application semantics.** The 14 tests form a deliberate progression:

1. **Tests 1–3**: Basic delivery — single message, sequential messages, concurrent proposals from different nodes. Establishes that the protocol works at all.
2. **Tests 4–6**: Fault tolerance — crash with majority surviving, recovery with catch-up, minority liveness impossibility. These test the Paxos quorum properties.
3. **Tests 7–8**: Consensus internals — competing proposers (ballot-number preemption) and decided-value immutability. Directly exercises the Paxos prepare/accept mechanism.
4. **Tests 9–10**: Linearizable register — write/read and compare-and-set built on top of TOB. Validates the "TOB = consensus = linearizability" equivalence from DDIA Chapter 9.
5. **Tests 11–14**: Ordering invariants — delivery callbacks fire in slot order, FIFO per-sender ordering holds, 25-message stress test, and replicated state machine convergence.

**Simulated network with round-based execution.** `cluster.run_until_delivered(n)` drives a synchronous simulation loop — no real networking, no threads. The optional `max_rounds` parameter (test 6) caps execution to prove liveness failure.

**Closure factory for callbacks.** Test 14 uses `make_cb(node_id)` to avoid the classic Python closure-over-loop-variable bug — each callback captures a distinct `node_id` rather than sharing a mutable loop variable.

## Dependencies

**Imports:**
- `pytest` — test framework
- `total_order_broadcast` — the implementation module (`ConsensusInstance`, `TOBNode`, `TOBCluster`, `LinearizableRegister`)

**Imported by:**
- `tester_test_total_order_broadcast.py` — likely a meta-test or auto-grader that validates this test file itself (consistent with the `tester_*` naming convention across all modules in the repo).

## Flow

Each test follows the same pattern:

1. **Create** a `TOBCluster(3)` — 3 nodes, requiring a majority of 2 for quorum.
2. **Broadcast** one or more messages from specific node IDs.
3. **Drive** the simulation with `run_until_delivered(expected_count)`, which runs message-passing rounds until all live nodes have delivered the expected number of messages (or `max_rounds` is hit).
4. **Assert** ordering via `verify_total_order()` (checks that all nodes agree on the same delivery sequence) and `get_delivery_order(node_id)` (returns the ordered list of delivered messages for a specific node).

For fault-tolerance tests, `crash_node(id)` / `recover_node(id)` are injected between broadcast phases. Recovery in test 5 asserts that the recovered node's delivery order matches the leader's — meaning catch-up replays missed slots.

For the linearizable register tests (9–10), `LinearizableRegister` wraps the cluster. Writes and CAS operations are broadcast as TOB messages; reads return the latest applied state. This is a textbook replicated state machine.

## Invariants

The tests collectively enforce these invariants:

1. **Total order**: All non-crashed nodes deliver the same messages in the same sequence (`verify_total_order()` in tests 1–3, 13–14).
2. **Validity**: Every broadcast message is eventually delivered (all `assert msg in order` checks).
3. **Quorum liveness**: A majority (2 of 3) can make progress; a minority (1 of 3) cannot (tests 4 vs 6).
4. **Ballot-number monotonicity**: A `prepare(n=2)` preempts a prior `prepare(n=1)`, so `accept(n=1)` is rejected (test 7).
5. **Decision immutability**: Once a consensus instance decides a value, `inst.is_decided` is true and `inst.decided_value` never changes (test 8).
6. **FIFO per sender**: Messages from the same sender are delivered in send order (test 12).
7. **Slot monotonicity**: Delivery callbacks fire with strictly increasing slot numbers `[0, 1, 2]` (test 11).
8. **State machine convergence**: All replicas applying the same commands in total order reach identical state (test 14 — all counters equal 10).

## Error Handling

The tests don't exercise error handling in the traditional sense. Instead:

- **Liveness failure** is tested explicitly: test 6 asserts that `run_until_delivered` returns `False` when quorum is lost, rather than hanging or raising.
- **CAS failure** is a semantic outcome, not an error: `compare_and_set("x", 10, 30)` returns `False` when the expected value doesn't match (test 10).
- **Consensus rejection**: `accept()` returns `{'accepted': False}` when the ballot number is stale (test 7) — this is the normal Paxos preemption path, not an exception.

No tests check for exception-throwing edge cases (e.g., broadcasting to a crashed node, invalid slot access). The protocol handles these internally via the quorum mechanism.

## Topics to Explore

- [file] `total-order-broadcast/total_order_broadcast.py` — The implementation: how `ConsensusInstance` implements Paxos prepare/accept, how `TOBCluster` simulates the network, and how `LinearizableRegister` applies commands
- [function] `total-order-broadcast/total_order_broadcast.py:TOBCluster.run_until_delivered` — The simulation driver: how it steps through rounds, delivers messages, and detects termination vs timeout
- [function] `total-order-broadcast/total_order_broadcast.py:ConsensusInstance.prepare` — The Paxos prepare phase: ballot-number tracking, promise semantics, and how previously accepted values are returned
- [general] `tob-linearizability-equivalence` — DDIA Chapter 9 proves that total-order broadcast and linearizability are equivalent in power; tests 9–10 demonstrate the "TOB → linearizable register" direction
- [file] `raft-consensus/raft.py` — Compare with Raft, another consensus protocol in this repo; Raft bundles leader election + log replication while this TOB module separates per-slot consensus instances

## Beliefs

- `tob-tests-enforce-total-order-across-all-live-nodes` — `verify_total_order()` asserts that every non-crashed node's delivery sequence is identical, which is the defining safety property of total-order broadcast
- `tob-quorum-is-strict-majority` — The test cluster uses 3 nodes; 2 can make progress (tests 4–5) but 1 alone cannot (test 6), confirming a strict-majority quorum requirement
- `tob-consensus-uses-paxos-ballot-numbers` — `ConsensusInstance.prepare(n)` / `accept(n, val)` use monotonic proposal numbers where a higher prepare preempts a lower one (test 7)
- `tob-linearizable-register-is-built-on-broadcast` — `LinearizableRegister` wraps `TOBCluster` to provide write/read/CAS, demonstrating the equivalence between total-order broadcast and linearizable storage (tests 9–10)
- `tob-recovery-replays-missed-slots` — After `recover_node(2)`, the recovered node's delivery order matches the live nodes' order, meaning recovery replays all consensus decisions made while the node was down (test 5)

