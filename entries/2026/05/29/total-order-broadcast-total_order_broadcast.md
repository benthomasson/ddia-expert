# File: total-order-broadcast/total_order_broadcast.py

**Date:** 2026-05-29
**Time:** 09:07

# `total-order-broadcast/total_order_broadcast.py`

## Purpose

This file implements **Total Order Broadcast** (TOB) — a distributed systems primitive that guarantees all nodes deliver messages in the same order. It's a reference implementation for Chapter 9 of *Designing Data-Intensive Applications*, which discusses consensus and total order broadcast as equivalent problems.

The module builds TOB on top of **Multi-Paxos**: each message slot is decided by an independent single-decree Paxos instance. It then demonstrates the practical payoff by building a `LinearizableRegister` (a linearizable key-value store) on top of the broadcast — the canonical reduction from consensus to linearizability.

## Key Components

### `ConsensusInstance` (lines 4–50)

A single-decree Paxos acceptor/learner for one slot in the log. Each instance tracks:

- **`promised`**: highest proposal number this acceptor has promised not to accept anything lower than (Phase 1 state).
- **`accepted_proposal` / `accepted_value`**: the highest-numbered proposal this acceptor has accepted (Phase 2 state).
- **`_decided` / `_decided_value`**: whether a value has been chosen for this slot.

Key contracts:
- `prepare(proposal_number, proposer_id)` → returns `{promised: bool, accepted_proposal, accepted_value}`. Promises only if `proposal_number > self.promised`.
- `accept(proposal_number, value, proposer_id)` → returns `{accepted: bool}`. Accepts only if `proposal_number >= self.promised`.
- `force_decide(value)` → unconditionally marks the slot as decided (used for catch-up, not consensus).

### `TOBNode` (lines 53–278)

The core protocol participant. Each node is simultaneously a Paxos **proposer**, **acceptor**, and **learner** for every slot. Key state:

- **`_instances`**: map of slot → `ConsensusInstance` (created lazily).
- **`_pending`**: FIFO queue of messages waiting to be proposed.
- **`_proposals`**: active Paxos rounds, keyed by slot. Each tracks round number, phase (`prepare`/`accept`/`done`), promise/accept vote counts, and the highest previously-accepted value (for Paxos's value-adoption rule).
- **`_delivered`**: ordered list of `(slot, message)` pairs — the final total-ordered log.
- **`_next_slot`**: the frontier of contiguous delivery. Messages are only delivered when all prior slots are decided.

Key methods:

| Method | Role |
|--------|------|
| `broadcast(message)` | Enqueues a message for ordering |
| `tick()` | Drives the protocol: delivers decided slots, starts proposals for pending messages, retries preempted proposals |
| `receive(msg)` | Dispatches incoming protocol messages to handlers |
| `_make_proposal_number(round_num)` | Generates globally unique proposal numbers: `round * N + node_id` |
| `_find_proposal_slot()` | Finds lowest undecided, uncontested slot |
| `_bump_and_retry(slot, outgoing)` | Increments the round and restarts Phase 1 after preemption |
| `crash()` / `recover(decided_slots)` | Simulates node failure and state-transfer recovery |

### `TOBCluster` (lines 281–353)

A simulated network that owns all nodes and routes messages synchronously. It provides the test harness:

- `run_until_delivered(expected_count, max_rounds)` — runs tick/route cycles until all alive nodes have delivered enough messages.
- `_route_messages(messages, depth)` — recursively delivers messages and collects responses within a single round, with a depth limit of 500 to prevent stack overflow.
- `verify_total_order()` — asserts all alive nodes have identical delivery sequences.

### `LinearizableRegister` (lines 356–432)

Demonstrates the reduction from TOB to linearizability. Each operation (`read`, `write`, `compare_and_set`) is broadcast as a message, and the result is determined by the global delivery order. This is the key insight from DDIA: if you can order all operations consistently, you get linearizability for free.

- Reads are **not** local — they go through the broadcast to establish a consistent point in the total order.
- CAS checks `current == expected` at delivery time, not submission time.
- Results are stashed in `_op_results` keyed by `op_id` and retrieved after `run_until_delivered` returns.

## Patterns

**Multi-Paxos via single-decree composition**: Rather than implementing Multi-Paxos directly with a stable leader, each slot runs independent Paxos. This is simpler but less efficient — every message triggers a full two-phase protocol.

**Proposal number encoding**: `round * num_nodes + node_id` guarantees uniqueness across nodes without coordination. Node 0's proposals are `0, 3, 6, ...`; node 1's are `1, 4, 7, ...` (for a 3-node cluster).

**Value adoption**: In `_handle_prepare_response`, if any acceptor reports a previously accepted value, the proposer *must* adopt the highest-numbered one — this is the core Paxos safety invariant that prevents decided values from being overwritten.

**Re-queuing on preemption**: When a proposer's value loses a slot (another value was decided), the original value is pushed back onto `_pending` to be proposed in a later slot. This ensures no messages are lost.

**Contiguous delivery**: `_deliver_decided_slots` only advances `_next_slot` through a contiguous run of decided slots. A gap (undecided slot 5 when 6 is decided) blocks delivery of slot 6 — this is what guarantees total order.

**Synchronous simulation**: `TOBCluster._route_messages` is recursive — it delivers all messages produced in one round before returning, simulating synchronous message passing. The depth limit prevents infinite loops during contention.

## Dependencies

**Imports**: None — the module is self-contained with no external dependencies.

**Imported by**:
- `test_total_order_broadcast.py` — unit/integration tests
- `tester_test_total_order_broadcast.py` — likely a meta-test or test validator

## Flow

A typical broadcast follows this path:

1. **Client calls** `node.broadcast("msg")` → message enters `_pending` queue.
2. **`tick()`** pops the message, finds an open slot via `_find_proposal_slot()`, starts Phase 1 by sending `prepare` to all nodes.
3. **Each node's `receive()`** handles the prepare → acceptor calls `inst.prepare()` → sends `prepare_response`.
4. **Proposer collects promises** in `_handle_prepare_response`. On majority: adopts any previously-accepted value, transitions to Phase 2, sends `accept_request` to all.
5. **Each node's acceptor** handles accept → `inst.accept()` → sends `accept_response`.
6. **Proposer collects accepts** in `_handle_accept_response`. On majority: marks instance decided, broadcasts `decided` to all, calls `_deliver_decided_slots()`.
7. **All nodes receive `decided`** → force-decide the instance, deliver contiguous slots, re-queue any displaced proposals.

**Contention path**: If a prepare or accept is rejected (a higher proposal number was seen), the proposer counts rejections. Once a majority is impossible, `_bump_and_retry` increments the round and restarts Phase 1.

## Invariants

- **Total order**: All alive nodes deliver messages in the same sequence. Enforced by Paxos consensus per slot + contiguous delivery.
- **No gaps in delivery**: `_next_slot` advances only through consecutive decided slots. A hole blocks everything behind it.
- **Paxos safety — value adoption**: A proposer that receives any previously-accepted values in Phase 1 *must* propose the one with the highest proposal number. This prevents overwriting a value that may already be decided.
- **Unique proposal numbers**: The `round * N + node_id` encoding guarantees no two nodes generate the same proposal number.
- **No message loss**: If a proposer's value loses its slot to a competing value, the original is re-queued in `_pending`.
- **Majority quorum**: Both Phase 1 and Phase 2 require `N/2 + 1` votes. Two majorities always overlap, ensuring any decided value is visible to future proposers.

## Error Handling

There is essentially none — this is a protocol simulation, not production code:

- **No network failures modeled**: Messages are delivered synchronously and reliably by `TOBCluster._route_messages`.
- **No timeouts**: Preemption is detected via rejection counts, not timeouts.
- **Crash/recovery** is explicit: `crash()` sets `_alive = False`, making the node drop all messages. `recover()` replays decided slots from a peer — there's no persistent storage or WAL.
- **`_route_messages` depth limit**: Caps recursion at 500 to prevent stack overflow during heavy contention, but remaining undelivered messages are silently deferred to the next tick rather than raising.
- **`run_until_delivered` timeout**: Returns `False` if `max_rounds` is exhausted — callers must check the return value.

---

## Topics to Explore

- [file] `total-order-broadcast/test_total_order_broadcast.py` — See how contention, crash/recovery, and linearizability are tested against this implementation
- [function] `total_order_broadcast.py:_handle_prepare_response` — The value-adoption logic is the crux of Paxos safety; trace what happens when two proposers compete for the same slot
- [general] `multi-paxos-vs-single-decree` — This implementation runs independent Paxos per slot; compare with a Multi-Paxos leader-lease optimization that skips Phase 1 for a stable leader
- [file] `raft-consensus/raft.py` — Compare this Paxos-based approach with the Raft implementation in the same repo; Raft bundles leader election with log replication rather than per-slot consensus
- [general] `linearizable-reads-via-tob` — The `LinearizableRegister.read()` broadcasts a read through consensus rather than reading locally; explore why this is necessary and when you can avoid it (e.g., leader leases)

## Beliefs

- `tob-paxos-per-slot` — Each slot in the total order log is decided by an independent single-decree Paxos instance; there is no stable-leader optimization
- `tob-contiguous-delivery` — Messages are delivered to the application only when all prior slots are decided; a gap at slot N blocks delivery of slot N+1 and beyond
- `tob-value-adoption-enforced` — During Phase 1, the proposer adopts the previously-accepted value with the highest proposal number, preserving Paxos safety
- `tob-proposal-numbers-unique` — Proposal numbers are encoded as `round * num_nodes + node_id`, guaranteeing global uniqueness without coordination
- `tob-linearizable-reads-go-through-consensus` — `LinearizableRegister.read()` broadcasts through the total order log rather than reading local state, ensuring linearizability at the cost of a full consensus round per read

