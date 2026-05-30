# Topic: How many sync rounds different topologies (ring, star, fully-connected) need for N nodes to converge

**Date:** 2026-05-29
**Time:** 13:56

## Convergence Rounds by Topology

### What the code actually implements: random peer selection (fully-connected)

This codebase implements **exactly one gossip topology**: random peer selection from the full membership. There is no ring, star, or structured overlay — every node can gossip with every other node.

The peer selection logic is in `gossip_protocol.py:170-172`:

```python
peers = [p for p in active_ids if p != nid]
if peers:
    peer_id = self.rng.choice(peers)
```

Each round, every active node picks **one random peer** and does a **bidirectional exchange** (lines 174-177). This means a single round produces up to N exchanges (one per node), and information can spread exponentially — each node that learns something becomes a potential source next round.

### The O(log N) convergence bound

The test `test_convergence_speed_olog_n` (`test_gossip_protocol.py:83`) empirically validates this for N = 8, 16, 32, 64:

```python
assert converged_round <= 5 * math.log2(n_nodes) + 5
```

This is the classic epidemic/gossip dissemination result: with random peer selection from a fully-connected graph and fanout=1, information reaches all N nodes in **O(log N) rounds**. The constant factor of 5 in the test is generous — in practice with bidirectional exchange, convergence is faster.

### Ring and star topologies: not implemented, but here's the theory

The codebase doesn't constrain peer selection to ring or star neighbors, so these numbers come from distributed systems theory, not from this code:

| Topology | Rounds to converge | Why |
|---|---|---|
| **Fully-connected** (this impl) | O(log N) | Exponential spread — informed nodes double each round |
| **Ring** | O(N) | Information travels one hop per round in each direction; worst case is N/2 hops (diameter) |
| **Star** (hub-and-spoke) | O(N) with 1 hub, but 2 rounds if hub fans out | All information must transit the hub; hub can only gossip with one peer per round unless fanout > 1 |

To implement ring or star topologies in this codebase, you'd need to replace the random peer selection at line 170 with a topology-aware neighbor list — for example, each node would only be allowed to gossip with its ring neighbors or hub node.

### Why the implementation avoids structured topologies

The `GossipCluster.gossip_round` method (line 140) is designed for **epidemic-style protocols** as described in DDIA — the whole point is that random peer selection gives you O(log N) convergence without maintaining any topology structure. Structured topologies trade convergence speed for reduced per-round message overhead (ring: 2 messages/node/round vs. fully-connected: 1 random but from N-1 candidates).

## Topics to Explore

- [function] `gossip-protocol/gossip_protocol.py:receive_gossip` — The merge logic that resolves conflicts during gossip exchange; understanding "higher heartbeat wins" is key to why convergence works at all
- [function] `gossip-protocol/gossip_protocol.py:detect_failures` — How timeout-based failure detection interacts with convergence speed; a slow-converging topology would cause false positives
- [general] `gossip-fanout-parameter` — This implementation uses fanout=1 (one random peer per round); increasing fanout to k multiplies message cost but reduces convergence to O(log_k N) — worth experimenting with
- [general] `ring-topology-gossip` — Implementing a ring overlay by constraining peer selection would let you empirically measure the O(N) vs O(log N) difference
- [file] `gossip-protocol/test_gossip_protocol.py` — The convergence speed test at line 83 is a good starting point for adding topology-aware convergence benchmarks

## Beliefs

- `gossip-topology-is-fully-connected` — GossipCluster uses unrestricted random peer selection (any node can gossip with any other), making the effective topology a fully-connected graph
- `gossip-fanout-is-one` — Each node selects exactly one random peer per gossip round, with bidirectional exchange producing at most N pairwise syncs per round
- `gossip-convergence-is-olog-n` — The implementation converges in O(log N) rounds, empirically validated up to N=64 with the bound 5*log2(N)+5
- `no-structured-topology-support` — The codebase has no ring, star, or mesh topology modes; peer selection is always uniform random from all active nodes

