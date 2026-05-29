# File: lamport-clocks/lamport.py

**Date:** 2026-05-29
**Time:** 08:57

# `lamport-clocks/lamport.py` — Lamport Logical Clocks

## Purpose

This file implements Lamport logical clocks and their primary application: distributed mutual exclusion. It's a reference implementation of two foundational algorithms from Leslie Lamport's 1978 paper "Time, Clocks, and the Ordering of Events in a Distributed System" — the logical clock protocol itself, and the mutex algorithm built on top of it. The file owns all event ordering logic: clock advancement, causal tracking via the happens-before relation, and total ordering of events across nodes.

## Key Components

### Data Classes

**`Event`** — Represents a single event (LOCAL, SEND, or RECEIVE) at a node. Beyond the visible fields (`node_id`, `event_type`, `timestamp`, `description`, `message_id`), it carries two hidden graph edges:
- `_parent`: links to the previous event on the same node (intra-process ordering)
- `_cause`: links a RECEIVE event back to the corresponding SEND event (inter-process causality)

These two edges form the happens-before graph that `_reaches()` traverses.

**`Message`** — A simple envelope carrying `sender_id`, `sender_timestamp`, `message_id`, and `payload`. The timestamp piggybacks on the message so the receiver can update its clock.

### `LamportClock`

A single integer counter with three operations:

- **`tick()`** — Increment for local events. Returns the new value.
- **`send_tick()`** — Identical to `tick()`. Exists as a separate method for semantic clarity (the Lamport protocol treats send as "increment then stamp").
- **`receive_tick(received_timestamp)`** — The core Lamport rule: `counter = max(local, received) + 1`. This is what ensures causal consistency — a receive event always has a timestamp strictly greater than the corresponding send.

### `Node`

A process in the distributed system. Each node owns a `LamportClock` and an append-only `_event_log`. The three event methods mirror the three clock operations:

- **`local_event(description)`** — Ticks the clock, appends a LOCAL event with `_parent` pointing to the previous event on this node.
- **`send_message(to_node, payload)`** — Ticks the clock, records a SEND event, creates a `Message`, and **delivers synchronously** by calling `to_node.receive_message()`. This is a simplification — real systems would queue and deliver asynchronously.
- **`receive_message(message, _send_event)`** — Applies `receive_tick`, records a RECEIVE event with both `_parent` (previous local event) and `_cause` (the send event that produced this message).

### `LamportMutex`

Implements Lamport's distributed mutex algorithm. The protocol:

1. **`request(node)`** — The requesting node ticks its clock, adds `(timestamp, node_id)` to the shared request queue, then broadcasts REQUEST messages to all other nodes. Each recipient immediately sends an ACK back. The mutex tracks which ACKs have been received per requesting node.

2. **`can_enter(node)`** — A node can enter the critical section when:
   - Its own request is at the head of the queue (lowest `(timestamp, node_id)`)
   - It has received ACKs from **all** other nodes

3. **`release(node)`** — Removes the node's request from the queue and broadcasts RELEASE to all others.

The request queue is sorted by `(timestamp, node_id)`, which gives a deterministic total order even when timestamps collide.

### Free Functions

**`total_order(events)`** — Sorts events by `(timestamp, node_id)`. This is the standard Lamport total order — timestamps provide causal ordering, node IDs break ties deterministically.

**`happens_before(event_a, event_b, all_events)`** — Determines the causal relationship: `True` if a→b, `False` if b→a, `None` if concurrent. Note that the `all_events` parameter is accepted but unused — the implementation relies entirely on the `_parent`/`_cause` graph edges.

**`_reaches(source, target)`** — BFS backward from `target` through `_parent` and `_cause` edges to check if `source` is reachable. This is the operational definition of happens-before: a→b iff there's a path from a to b in the causal graph.

## Patterns

- **Causal graph via hidden fields**: The `_parent` and `_cause` fields on `Event` are excluded from `repr` (via `field(repr=False)`) and form an implicit DAG. This separates the causal structure (used by `happens_before`) from the observable event data.

- **Synchronous message delivery**: `send_message` calls `receive_message` directly, making the simulation deterministic and single-threaded. The `_send_event` parameter threads the causal link through the delivery.

- **Global message counter**: `_msg_counter = itertools.count(1)` is module-level state that generates unique message IDs across all nodes. This avoids coordination but means message IDs are tied to execution order.

- **Centralized mutex state**: The `LamportMutex` holds the request queue and ACK tracking in a single object, simulating what would be replicated state in a real distributed system. Each node would maintain its own copy of the queue in practice.

## Dependencies

**Imports**: Standard library only — `dataclasses`, `typing`, `collections.deque`, `itertools`. No external dependencies.

**Imported by**: `test_lamport.py` — the test suite that exercises clock behavior, message passing, mutex correctness, and happens-before reasoning.

## Flow

A typical interaction:

1. Create several `Node` instances
2. Nodes perform `local_event()` calls (clock ticks: 1, 2, 3...)
3. Node A calls `send_message(B, "hello")`:
   - A's clock ticks (e.g., to 4), SEND event recorded
   - `Message(sender_id="A", sender_timestamp=4, ...)` created
   - B's `receive_message` called immediately
   - B's clock becomes `max(B_clock, 4) + 1`, RECEIVE event recorded with `_cause` pointing to A's SEND event
4. `total_order()` can sort all events from all nodes into a single consistent sequence
5. `happens_before()` can determine if any two events are causally related by walking the `_parent`/`_cause` graph

For mutual exclusion, the flow adds:

1. Wrap nodes in a `LamportMutex`
2. `mutex.request(node)` broadcasts REQUEST, collects ACKs
3. `mutex.can_enter(node)` checks queue head + ACK completeness
4. After critical section work, `mutex.release(node)` broadcasts RELEASE

## Invariants

- **Lamport clock monotonicity**: A node's clock never decreases. `tick()` and `send_tick()` always increment; `receive_tick()` takes `max + 1`, which is always ≥ current + 1.
- **Causal consistency**: If event a causally precedes event b, then `a.timestamp < b.timestamp`. This is guaranteed by `receive_tick` using `max(local, sender) + 1`.
- **The converse does NOT hold**: `a.timestamp < b.timestamp` does not imply a→b. Two concurrent events can have ordered timestamps. This is a fundamental limitation of Lamport clocks (vector clocks address it).
- **Mutex safety**: `can_enter` requires both (1) the node's request is the queue minimum and (2) all other nodes have ACKed. Together these guarantee mutual exclusion — at most one node can satisfy both conditions simultaneously.
- **Total order determinism**: Ties in timestamp are broken by `node_id` string comparison, giving a total order consistent across all nodes.

## Error Handling

There is essentially none. The code assumes:
- Nodes passed to `send_message` are valid and reachable
- Messages are never lost or duplicated (synchronous delivery guarantees this)
- `receive_tick` always receives a non-negative integer
- The mutex is used correctly (request before can_enter, release after entering)

No exceptions are raised or caught anywhere. This is appropriate for a reference implementation — it models the algorithm's happy path, not network failure modes.

## Topics to Explore

- [file] `vector-clocks/vector_clock.py` — The natural successor to Lamport clocks; vector clocks can distinguish concurrent events from causally ordered ones, fixing the "converse doesn't hold" limitation
- [file] `lamport-clocks/test_lamport.py` — See how the happens-before relation and mutex are tested; reveals edge cases the implementation handles
- [function] `lamport-clocks/lamport.py:_reaches` — The BFS graph traversal is the computational core of happens-before detection; understanding its performance characteristics matters for large event histories
- [general] `lamport-1978-paper` — Lamport's original "Time, Clocks, and the Ordering of Events in a Distributed System" paper, which this code directly implements
- [file] `total-order-broadcast/total_order_broadcast.py` — Total order broadcast builds on the same clock-based ordering ideas to deliver messages in a consistent sequence across nodes

## Beliefs

- `lamport-receive-tick-guarantees-causal-order` — `receive_tick` computes `max(local, received) + 1`, ensuring every receive event has a timestamp strictly greater than the corresponding send event
- `lamport-happens-before-uses-graph-not-timestamps` — `happens_before()` determines causality by BFS over `_parent`/`_cause` edges, not by comparing timestamps; the `all_events` parameter is accepted but never used
- `lamport-mutex-requires-all-acks` — `can_enter()` returns True only when the node's request is at the queue head AND acknowledgments have been received from every other node in the system
- `lamport-send-delivers-synchronously` — `send_message` calls `to_node.receive_message()` directly in the same call stack, making message delivery instantaneous and deterministic (no async queue or network simulation)
- `lamport-total-order-breaks-ties-by-node-id` — `total_order()` sorts by `(timestamp, node_id)`, using lexicographic node ID comparison as a deterministic tiebreaker when timestamps are equal

