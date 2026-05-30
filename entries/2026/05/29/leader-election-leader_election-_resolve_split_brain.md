# Function: _resolve_split_brain in leader-election/leader_election.py

**Date:** 2026-05-29
**Time:** 13:21

## `_resolve_split_brain` — Split-Brain Resolution in Bully Leader Election

### Purpose

This is a safety net on the `BullyElectionCluster` simulation harness that detects and resolves **split-brain conditions** — situations where two or more nodes simultaneously believe they are the leader. In a real distributed system, split-brain is one of the most dangerous failure modes because conflicting leaders can accept divergent writes. This method enforces the Bully Algorithm's core invariant: **the highest-ID available node is always the rightful leader**.

It exists because the message-passing simulation in `tick()` can leave the cluster in a transiently inconsistent state. Network partitions healing, simultaneous elections, or message delivery ordering can all produce multiple self-proclaimed leaders within a single tick. Rather than proving the message loop is always convergent in one pass, the code takes the pragmatic route of detecting and correcting the violation after each tick completes.

### Contract

- **Precondition**: Called after all tick messages have been fully delivered (the `while all_messages` loop in `tick()` has drained). The cluster's message-passing phase is complete, so node states reflect the outcome of this tick's communication.
- **Postcondition**: At most one node remains in `"leader"` state. All other former leaders have transitioned to `"candidate"` and their election messages have been delivered (one hop of responses processed).
- **Invariant enforced**: The surviving leader is always `max(leaders)` — the highest-ID node among those claiming leadership. This is the Bully Algorithm's fundamental rule.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `current_time` | `int` | The simulation's logical clock value for the current tick. Passed through to `start_election()` so that election messages carry the correct timestamp. |

No validation is performed on `current_time`. The method assumes it's a monotonically increasing integer consistent with the rest of the simulation.

### Return Value

None. This method operates entirely through side effects.

### Algorithm

1. **Detect**: Scan all nodes and collect IDs of those that are both available (`is_available() == True`) and in `"leader"` state. Unavailable nodes are excluded — a crashed node's stale state doesn't count.

2. **Guard**: If zero or one leaders exist, there's no split-brain. Exit immediately.

3. **Choose winner**: `highest = max(leaders)` — the highest-ID leader survives. This node is left completely untouched; it keeps its `"leader"` state, term, and heartbeat schedule.

4. **Force losers to step down**: For each other leader (`lid != highest`):
   - Call `node.start_election(current_time)`, which:
     - Increments the node's term
     - Sets its state to `"candidate"`
     - Clears its `leader_id`
     - Returns `ELECTION` messages addressed to all higher-ID nodes
   - This effectively demotes the node from leader to candidate.

5. **Deliver one round of messages**: The `ELECTION` messages are delivered to their receivers. Each receiver may respond (e.g., with `ALIVE` if it has a higher ID, or by starting its own election). Those responses are delivered too — but only one level deep. The delivery is **not** recursive; it processes exactly two hops (election → response → delivery of response).

### Side Effects

- **Node state mutations**: Lower-ID leaders have their `_state`, `_leader_id`, `_current_term`, `_election_start_time`, and `_got_alive` fields modified by `start_election()`.
- **Message receivers' state**: Nodes that receive the `ELECTION` messages may change their own state (e.g., a higher-ID follower receiving `ELECTION` will respond with `ALIVE` and potentially start its own election).
- **No history recording**: Unlike the main `tick()` loop, this method does not call `_record_leader()`, so elections triggered by split-brain resolution are not captured in `election_history`. This is a subtle gap — if a split-brain resolution triggers a new `COORDINATOR` message that results in a leadership change, it won't appear in the history.

### Error Handling

None. The method silently handles missing receivers via `self.nodes.get(msg.receiver_id)` returning `None` and the `if receiver` guard skipping delivery. No exceptions are raised or caught. If a node ID in a message doesn't exist in `self.nodes`, the message is silently dropped.

### Usage Patterns

Called exactly once per `tick()`, as the final step after all normal message delivery has completed:

```python
def tick(self, current_time: int) -> None:
    # ... collect and deliver all messages ...
    self._resolve_split_brain(current_time)
```

This is a private method (`_` prefix) — it's internal to `BullyElectionCluster` and not part of the public API. Callers of `tick()` don't need to know it exists; they just get the guarantee that split-brain won't persist across ticks.

### Dependencies

- **`BullyNode.is_available()`** — to filter out crashed nodes
- **`BullyNode.state`** — property to read the node's current role
- **`BullyNode.start_election()`** — to demote a node and generate election messages
- **`BullyNode.receive_message()`** — to deliver election messages and process responses

### Assumptions Not Enforced by Types

1. **`current_time` is consistent with `self._current_time`** — nothing prevents passing a stale or future timestamp.
2. **Two hops of delivery is sufficient** — the code delivers election messages and their immediate responses, but if those responses trigger further cascading messages, they are dropped. This assumes one round is enough to begin convergence; the next `tick()` will finish the job.
3. **`max(leaders)` is the correct winner** — this hard-codes the Bully Algorithm's rule. If the election protocol were changed (e.g., to Raft's term-based leadership), this resolution logic would silently enforce the wrong invariant.
4. **No concurrent modification** — the method mutates node state while iterating, assuming single-threaded execution. In a concurrent environment, the leader list could change mid-iteration.

## Topics to Explore

- [function] `leader-election/leader_election.py:BullyNode.start_election` — The demotion mechanism: how a node transitions from leader to candidate and what messages it emits
- [function] `leader-election/leader_election.py:BullyElectionCluster.tick` — The full tick cycle that calls `_resolve_split_brain` as its final step; understanding message delivery ordering explains why split-brain can occur
- [function] `leader-election/leader_election.py:BullyElectionCluster.recover_node` — Uses the same inline message-delivery pattern; compare the two to understand the duplication
- [file] `leader-election/test_leader_election.py` — Test cases that exercise split-brain scenarios, partition healing, and multi-failure recovery
- [general] `bully-algorithm-vs-raft` — How Bully's "highest ID wins" compares to Raft's term-based consensus, and why Bully needs an external split-brain resolver while Raft's protocol prevents it structurally

## Beliefs

- `split-brain-highest-id-wins` — `_resolve_split_brain` always keeps the highest-ID leader and demotes all others, enforcing the Bully Algorithm invariant
- `split-brain-runs-after-message-drain` — Split-brain resolution executes only after the tick's message delivery loop has fully drained, never mid-delivery
- `split-brain-delivers-two-hops` — Election messages from demoted leaders are delivered one level deep (election + response), not recursively to convergence
- `split-brain-not-recorded-in-history` — Elections triggered by `_resolve_split_brain` do not call `_record_leader`, so leadership changes from split-brain resolution are absent from `election_history`
- `split-brain-ignores-unavailable-nodes` — Only nodes where `is_available()` is `True` are considered when detecting and resolving split-brain; crashed nodes' stale leader state is ignored

