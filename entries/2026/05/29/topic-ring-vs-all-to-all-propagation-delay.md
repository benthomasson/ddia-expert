# Topic: In all-to-all topology, changes reach every node in one hop (no re-queueing needed for coverage); in ring topology, propagation takes N-1 hops — explore how this affects convergence latency and conflict probability

**Date:** 2026-05-29
**Time:** 14:15

Now I have enough context from both the multi-leader replication and gossip protocol modules. Here's the explanation.

---

# All-to-All vs Ring Topology: Convergence Latency and Conflict Probability

## The Core Mechanism

The multi-leader replication module (`multi-leader-replication/multi_leader.py`) implements both topologies explicitly, and the difference comes down to one question: **who does each node send its pending changes to during a sync round?**

### All-to-All: One Hop, Full Coverage

In `MultiLeaderCluster.sync()` with `Topology.ALL_TO_ALL` (line ~57 of `multi_leader.py`), each source node's pending changes are delivered to **every other node** in a single round. The algorithm:

1. Drain `_pending` from all nodes via `get_pending_changes()` (snapshot isolation — nothing generated during delivery contaminates this round).
2. For each source node's changes, call `apply_remote_change()` on every other node.

A change from node A reaches nodes B, C, D, and E simultaneously. **One `sync()` call suffices for convergence** when there are no custom-merge cascades. The test at `test_multi_leader.py:5-18` (`test_lww`) demonstrates this: two concurrent writes, one `sync()`, then `all_converged()` is true.

### Ring: Store-and-Forward, N-1 Hops

With `Topology.RING`, each node sends only to its successor at position `(i+1) % len(_node_order)`. A change originating at node 0 in a 3-node ring takes this path:

- **Round 1**: Node 0's change reaches node 1.
- **Round 2**: Node 1 (which received the change in round 1 and re-enqueued it in its `_pending` via `apply_remote_change` at line ~117) forwards it to node 2.

This works because `apply_remote_change` has a critical side effect: when a node accepts a new change, it appends the change dict to its own `_pending` list (line ~117 and ~90 of `multi_leader.py`). The next `sync()` call drains this re-enqueued change and sends it along to the next neighbor.

The test at `test_multi_leader.py:39-43` (`test_ring`) explicitly asserts this:

```python
rounds = cluster.sync_until_converged()
assert rounds >= 2, f'Expected >=2 rounds, got {rounds}'
```

For N nodes, convergence requires **at least N-1 rounds** — the change must hop through every intermediate node.

## Convergence Latency

| Topology | Rounds to Converge | Latency Model |
|----------|-------------------|---------------|
| ALL_TO_ALL | 1 (no cascading merges) | O(1) rounds |
| RING | N-1 | O(N) rounds |

The snapshot isolation property of `sync()` matters here. Since all pending queues are drained **before** any deliveries happen, a change can only advance one hop per round in ring topology. This is documented in the `MultiLeaderCluster` entry: "a change propagated in round N cannot cascade further until round N+1."

**Contrast with gossip**: The gossip protocol (`gossip_protocol.py`) uses a different approach entirely — **random pairwise exchange** (line ~164-172). Each active node picks a random peer and they do a bidirectional exchange. This gives O(log N) convergence, as validated by `test_convergence_speed_olog_n` (line ~86 of `test_gossip_protocol.py`), which asserts convergence within `5 * log2(N) + 5` rounds for clusters up to 64 nodes.

So we have three convergence profiles in this codebase:
- **All-to-all multi-leader**: O(1) — deterministic, one round
- **Random gossip**: O(log N) — probabilistic, epidemic spread
- **Ring multi-leader**: O(N) — deterministic, sequential hops

## Conflict Probability

Topology doesn't change *whether* conflicts occur — that depends on concurrent writes to the same key. But it profoundly affects **where and when** conflicts are detected.

### All-to-All: Conflicts Are Localized

When node A and node B both write to key `k` before a sync, every other node receives both writes in the same `sync()` round. Each node resolves the conflict independently using the same deterministic rule — `(timestamp, node_id)` tuple comparison for LWW (line ~149 of `multi_leader.py`). Because the comparison is deterministic and doesn't depend on arrival order, all nodes converge to the same winner.

The `test_lww_tiebreak` test (line ~72-80) demonstrates this: nodes `a` and `b` write the same key with the same timestamp, and `b` wins everywhere because `"b" > "a"` lexicographically.

### Ring: Cascading Conflicts and Ordering Hazards

In ring topology, the same concurrent-write scenario plays out differently:

1. Node 0 writes `k=X`, node 2 writes `k=Y`.
2. In round 1: Node 0's change reaches node 1, node 2's change reaches node 0.
3. Node 0 now sees a conflict (its local `X` vs incoming `Y`). It resolves it.
4. In round 2: Node 1 receives node 2's change (forwarded through node 0). But node 1 already has node 0's version from round 1. Another conflict resolution occurs.

Each intermediate node in the ring potentially sees and resolves the same conflict independently. The `_seen` set (checked at `apply_remote_change` line ~101) prevents duplicate application, but the **order** in which conflicting writes arrive at each node varies by position in the ring.

For LWW, arrival order doesn't matter — the tuple comparison is commutative. But for `CUSTOM_MERGE`, the situation is more complex. The merge function at line ~167-172 creates a **synthetic timestamp** (`max(local_ts, remote_ts) + 1`) and a **canonical origin** (`max(local_origin, remote_node)`). This synthetic change then re-enters the ring and may trigger further merge operations at downstream nodes.

### The Conflict Window

The key insight: **longer convergence time = wider conflict window**. During the N-1 rounds that ring topology takes to converge, additional writes can land on nodes that haven't yet received prior changes. In all-to-all, the window is exactly one round — after that single `sync()`, every node has seen everything.

This is why `sync()` returns the number of `apply_remote_change` calls (including idempotent skips via `_seen`), not the number of accepted changes — it lets you observe that ring topology generates more replication traffic per logical write.

## The Gossip Protocol as a Middle Ground

The gossip protocol's random peer selection (`gossip_protocol.py:164-172`) is neither all-to-all nor ring. It achieves a balance:

- **Faster than ring**: O(log N) vs O(N) because each round doubles the number of informed nodes (epidemic spread).
- **Cheaper than all-to-all**: Each node contacts only one peer per round, not N-1 peers.
- **No explicit conflict resolution**: Gossip uses **heartbeat counter monotonicity** (`receive_gossip` at line ~55-78) rather than LWW or custom merge. The "latest" information always wins, which is safe for membership state but wouldn't work for arbitrary key-value conflicts.

The trade-off: gossip convergence is probabilistic. The `test_convergence_speed_olog_n` test uses a generous bound (`5 * log2(N) + 5`) because the exact round depends on which peers happen to be selected.

## Summary

| Property | All-to-All | Ring | Gossip (random pair) |
|----------|-----------|------|---------------------|
| Convergence rounds | 1 | N-1 | O(log N) |
| Network cost per round | O(N²) changes | O(N) changes | O(N) exchanges |
| Conflict window | 1 round | N-1 rounds | O(log N) rounds |
| Conflict resolution | Deterministic, same round | Deterministic, but cascading | Monotonic counter (no conflicts) |
| Re-queueing needed? | No | Yes (`_pending`) | No (full state exchange) |

---

## Topics to Explore

- [function] `multi-leader-replication/multi_leader.py:apply_remote_change` — The heart of conflict detection and the `_pending` re-enqueue that enables ring propagation; trace through the LWW vs CUSTOM_MERGE branches
- [function] `gossip-protocol/gossip_protocol.py:receive_gossip` — Compare the heartbeat-monotonicity merge rule (line 55-78) with multi-leader's `_seen`-based idempotency; both prevent stale data from overwriting fresh state, but through different mechanisms
- [general] `custom-merge-cascade-in-ring` — What happens when CUSTOM_MERGE generates a synthetic change with a new timestamp that itself conflicts at the next hop? Whether this can cause convergence to exceed N-1 rounds
- [file] `vector-clocks/vector_clock.py` — Scalar Lamport clocks can't distinguish concurrent from causally ordered writes; vector clocks would let the system detect true concurrency and avoid false conflicts in ring propagation
- [general] `gossip-vs-anti-entropy` — This codebase's gossip uses random pairwise exchange (rumor mongering); anti-entropy protocols periodically reconcile full state, which guarantees convergence at the cost of bandwidth

## Beliefs

- `all-to-all-converges-in-one-round` — `ALL_TO_ALL` topology delivers every pending change to every other node in a single `sync()` call, achieving convergence in one round when no custom-merge cascades occur
- `ring-requires-n-minus-one-rounds` — `RING` topology advances changes by exactly one hop per `sync()` round due to snapshot-isolated pending queue draining, requiring N-1 rounds for a single-source change to reach all N nodes
- `ring-requeues-via-apply-remote-change` — `apply_remote_change` appends accepted changes to the receiving node's `_pending` list, enabling store-and-forward propagation in ring topology; without this, changes would stop at the first hop
- `topology-does-not-change-conflict-outcome` — Both topologies use the same deterministic `(timestamp, node_id)` comparison for LWW resolution; topology affects when and where conflicts are detected, not which value wins
- `longer-convergence-widens-conflict-window` — Ring topology's O(N) convergence time means additional writes can arrive during propagation, increasing the chance of further conflicts compared to all-to-all's single-round window

