# Topic: The default of 150 vnodes is a significant design choice — explore how it affects load balance variance vs. memory/computation cost

**Date:** 2026-05-29
**Time:** 12:56

# The 150-Vnode Default: Load Balance vs. Resource Cost

## The Core Tradeoff

The default of `num_vnodes=150` at `consistent_hashing.py:18` sits at a specific point on a curve that every consistent hashing implementation must navigate: **more virtual nodes means better load distribution, but each vnode costs memory, increases ring mutation time, and slows lookup slightly.**

## Why Load Balance Improves with More Vnodes

Each physical node gets `num_vnodes * weight` positions on the ring (`consistent_hashing.py:31`). With only 1 vnode per node, each node owns a single contiguous arc — and because MD5 hashing doesn't produce perfectly uniform spacing, those arcs vary wildly in size. With 150 vnodes, each node's "territory" is fragmented into 150 small arcs scattered around the ring. The law of large numbers kicks in: the total arc length per node converges toward the ideal 1/N fraction.

The test at `test_consistent_hashing.py:149-163` makes this explicit:

```python
def test_vnode_count_affects_balance():
    for vnodes in [1, 10, 50, 150, 500]:
        r = ConsistentHashRing(num_vnodes=vnodes)
        r.add_node("A"); r.add_node("B"); r.add_node("C")
        imbalances.append(r.load_imbalance())
    assert imbalances[-1] < imbalances[0]
```

And the load balance test at line 47 asserts that with 150 vnodes and 3 nodes, `load_imbalance() < 1.5` — meaning the most-loaded node holds less than 1.5× the average. That's a practical guarantee: no node gets more than 50% above its fair share.

## The Statistical Model

For `N` physical nodes each with `V` vnodes, you have `N * V` points on a 2³² ring. The standard deviation of a node's load fraction is approximately `1 / sqrt(N * V)`. With 3 nodes and 150 vnodes, that's `1 / sqrt(450) ≈ 0.047`, meaning each node's share fluctuates by roughly ±5% around the ideal 33.3%. At `V=10`, it's `1 / sqrt(30) ≈ 0.18` — ±18% fluctuation, which is the difference between a tolerable and an intolerable hotspot in production.

## The Memory and Computation Costs

Each vnode consumes two entries: one in `_ring_positions` and one in `_ring_nodes` (lines 21-22). These are Python lists, so each entry is a pointer (8 bytes) plus the object overhead. For 100 physical nodes × 150 vnodes = 15,000 entries, that's roughly 240KB of list storage — negligible on a server, but it scales linearly.

The real cost is **ring mutation time**. `add_node` at line 28 loops `vnode_count` times, and each iteration does a `bisect.bisect_left` (O(log n)) followed by a `list.insert` (O(n)) on the sorted positions list. For 150 vnodes on a ring already holding 15,000 entries, that's 150 × O(15,000) = ~2.25M element shifts. Similarly, `remove_node` at line 48 collects all indices and pops them in reverse — each pop is O(n).

**Lookup** (`get_node` at line 68) is a single `bisect.bisect` — O(log(N×V)). Going from 150 to 500 vnodes only adds ~1.7 more binary search steps. Lookup cost is not the bottleneck.

## Why 150 and Not 256 or 50?

The curve of diminishing returns flattens sharply. Going from 50 → 150 vnodes cuts variance by ~√3 ≈ 1.7×. Going from 150 → 500 cuts it by another ~√3.3 ≈ 1.8×, but you're already at <1.5 imbalance with 150 at 3 nodes, and at larger cluster sizes the improvement matters even less. Meanwhile, ring mutation cost triples. The 150 default occupies the "knee" of the curve — past the steep improvement zone, before the flat plateau where extra vnodes burn CPU for negligible gain.

This is also roughly where production systems land: Cassandra uses 256 tokens (vnodes) by default, Riak used 64 by default but on much larger rings. The implementation's choice of 150 is in the same neighborhood.

## Interaction with Weighted Nodes

The weight multiplier at line 31 (`int(self.num_vnodes * weight)`) means a weight=3.0 node gets 450 vnodes. The test at `test_consistent_hashing.py:91-96` shows this produces a load ratio between 2.0× and 4.5× — not exactly 3× because hashing variance applies to both nodes, and the small node (100 vnodes) has higher relative variance than the large one (300 vnodes). Higher base `num_vnodes` would tighten that ratio toward the true weight, but again at the cost of more ring entries.

## Topics to Explore

- [function] `consistent-hashing/consistent_hashing.py:get_load_distribution` — Computes per-node ring ownership by summing arc lengths; understanding this is key to reasoning about imbalance empirically
- [function] `consistent-hashing/consistent_hashing.py:add_node` — The transfer-map computation reveals what happens operationally when a node joins: which key ranges migrate, and how vnode count determines the granularity of those transfers
- [general] `jump-consistent-hashing` — An alternative algorithm (Lamping & Veach) that achieves perfect balance with O(1) memory per node but doesn't support arbitrary node removal — worth contrasting with the vnode approach
- [file] `hinted-handoff/hinted_handoff.py` — Uses `hash(key) % N` (line 91) instead of a vnode ring, showing how the same codebase uses a simpler partitioning scheme where consistent hashing's minimal-disruption property isn't needed
- [function] `consistent-hashing/consistent_hashing.py:get_nodes` — The preference list walk (line 88-95) shows how replication interacts with vnodes: with too few vnodes per node, the walk may need to traverse many ring positions before finding RF distinct physical nodes

## Beliefs

- `vnode-150-guarantees-sub-1.5-imbalance` — With 3 equally-weighted nodes and 150 vnodes each, `load_imbalance()` is asserted to stay below 1.5 (`test_consistent_hashing.py:49`)
- `vnode-count-scales-with-weight` — A node's actual vnode count is `int(num_vnodes * weight)`, so weighted nodes get proportionally more ring positions (`consistent_hashing.py:31`)
- `add-node-mutation-cost-is-quadratic-in-ring-size` — Each `add_node` call performs `vnode_count` list insertions into a sorted list, each O(total ring entries), making node addition O(V × N×V) in the worst case
- `lookup-cost-is-logarithmic-in-total-vnodes` — `get_node` performs a single `bisect.bisect` over `_ring_positions`, so key lookup is O(log(N×V)) regardless of cluster size (`consistent_hashing.py:73`)
- `more-vnodes-monotonically-improves-balance` — The test `test_vnode_count_affects_balance` asserts that imbalance at 500 vnodes is strictly less than at 1 vnode, and the statistical model (variance ∝ 1/V) guarantees monotonic improvement

