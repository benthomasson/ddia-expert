# Topic: How production systems (Cassandra) replace the fixed-threshold approach here with a probabilistic one that adapts to network conditions

**Date:** 2026-05-29
**Time:** 13:07

# From Fixed Thresholds to Probabilistic Failure Detection

## The Fixed-Threshold Approach in This Codebase

The gossip protocol implementation uses a simple three-threshold state machine for failure detection. Look at `gossip-protocol/gossip_protocol.py:10`:

```python
def __init__(self, node_id: str, t_suspect: int = 5, t_dead: int = 10, t_cleanup: int = 20):
```

These three constants define rigid time boundaries. The actual decision logic lives in `detect_failures` at line 76, which compares elapsed time since the last heartbeat update against these fixed cutoffs:

```python
elif elapsed > self.t_dead and info["status"] != "dead":       # line 93
    info["status"] = "dead"
elif elapsed > self.t_suspect and info["status"] == "alive":   # line 96
    info["status"] = "suspected"
```

The state transitions are deterministic: `alive → suspected → dead → removed`. Every node uses identical thresholds (`t_suspect=5`, `t_dead=10`, `t_cleanup=20`), and these never change at runtime. You can see this rigidity propagated through `GossipCluster.__init__` at line 117, where the same values are stamped onto every node created via `add_node` at line 127.

## Why Fixed Thresholds Break in Production

This approach has a fundamental problem: **it can't distinguish between "the network is slow right now" and "that node is actually dead."**

Consider a data center experiencing a GC pause or a network partition that adds 2 seconds of latency. With `t_suspect=5`, a node whose heartbeats normally arrive every 1 second will be marked suspected after just 3 missed heartbeats — even though it's perfectly healthy and the network will recover in moments. Conversely, in a fast, stable network, waiting a full 5 seconds before suspecting a node wastes time you could spend re-routing traffic.

The tests in `gossip-protocol/tester_test_gossip_protocol.py:20` confirm this brittle coupling — they must hardcode specific threshold values (`t_suspect=3, t_dead=6, t_cleanup=12`) and then carefully count rounds to hit the exact transition points.

## How Cassandra Solves This: The Phi Accrual Failure Detector

Cassandra replaces the binary "alive or suspected" check with a **continuous suspicion level** called **phi (φ)**. Instead of asking "has it been longer than `t_suspect`?", it asks "given the historical distribution of inter-arrival times for this node's heartbeats, how surprising is the current silence?"

### The key differences:

**1. It maintains a sliding window of heartbeat inter-arrival times per peer.** Where this implementation tracks only the latest timestamp (`timestamp_last_updated` at line 16) and a monotonic counter (`heartbeat_counter`), Cassandra keeps the last ~1000 inter-arrival intervals and fits them to an exponential (or normal) distribution.

**2. Suspicion is a continuous value, not a binary state.** The phi value is calculated as:

```
φ = -log₁₀(1 - CDF(timeSinceLastHeartbeat))
```

Where CDF is the cumulative distribution function of the observed inter-arrival times. A φ of 1 means there's a 10% chance this silence is normal. A φ of 3 means 0.1%. A φ of 8 means the probability of a healthy node being this quiet is one in a hundred million.

**3. The threshold adapts automatically.** A node with jittery heartbeats (high variance) gets a wider distribution, so the same silence duration produces a lower φ. A node with rock-steady heartbeats produces a high φ quickly when it goes quiet — because silence is genuinely surprising.

**4. Different callers can use different conviction thresholds.** The gossip protocol might act at φ > 8 (very conservative), while a read-repair mechanism might start rerouting at φ > 5. In the fixed-threshold model, there's only one `t_suspect` for everyone.

### What this would look like replacing `detect_failures`

Instead of the current logic at lines 86–98 where `elapsed > self.t_suspect` triggers a state change, a phi-accrual detector would:

1. Record every heartbeat arrival time in a bounded window (not just overwrite `timestamp_last_updated`)
2. Compute mean and variance of inter-arrival intervals
3. Calculate φ from the elapsed time and that distribution
4. Let the caller decide what φ level warrants action

The `receive_gossip` method (line 52) would need to feed arrival times into the distribution rather than just updating `timestamp_last_updated` at line 62. The three fixed thresholds (`t_suspect`, `t_dead`, `t_cleanup`) would collapse into a single configurable conviction level (e.g., `phi_convict_threshold = 8`), but the actual sensitivity would be driven by observed network behavior.

### What this codebase is missing to implement it

This implementation has no concept of inter-arrival time history — it only stores the most recent timestamp. There's no statistical modeling, no per-peer distribution tracking, and no continuous suspicion score. The entire failure detection path is a simple elapsed-time comparison against constants. That's the pedagogical point: this code teaches the protocol structure (gossip exchange, state machine, membership management), while Cassandra layers statistical sophistication on top of that same structure.

---

## Topics to Explore

- [general] `phi-accrual-failure-detector-paper` — The original 2004 paper by Hayashibara et al. defines the mathematical foundation; understanding the CDF-based phi calculation is essential to seeing why it adapts where fixed thresholds can't
- [function] `gossip-protocol/gossip_protocol.py:receive_gossip` — This is where heartbeat arrival times would be recorded into a sliding window; currently it only overwrites the latest timestamp, discarding the distribution information that phi needs
- [function] `gossip-protocol/gossip_protocol.py:detect_failures` — The fixed-threshold decision point that would be replaced by a phi calculation; compare its three `if/elif` branches against a single `φ > threshold` check
- [file] `leader-election/leader_election.py` — Uses a similar fixed `election_timeout` (line 27) for leader failure detection; the same phi-accrual approach could replace it, showing this isn't just a gossip problem but a general distributed systems pattern
- [file] `hinted-handoff/hinted_handoff.py` — Downstream consumer of failure detection results; understanding how hinted handoff routes around failures (line 130–145) shows why accurate detection matters — false positives trigger unnecessary hint storage and sloppy quorum overhead

## Beliefs

- `gossip-fixed-three-threshold-fsm` — Failure detection uses exactly three fixed time thresholds (`t_suspect`, `t_dead`, `t_cleanup`) producing a linear state machine `alive → suspected → dead → removed`, with no adaptation to network conditions
- `gossip-no-arrival-history` — Each node stores only the most recent `timestamp_last_updated` per peer, discarding all inter-arrival time history that would be needed for probabilistic failure detection
- `gossip-uniform-thresholds` — All nodes in a `GossipCluster` receive identical threshold values from the cluster constructor (line 127), making it impossible for individual nodes to calibrate sensitivity to their specific network path
- `gossip-detect-failures-is-stateless` — `detect_failures` makes decisions using only the current time minus `timestamp_last_updated`; it consults no historical distribution, making it a pure point-in-time comparison rather than a statistical inference

