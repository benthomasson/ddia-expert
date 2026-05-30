# Function: ConsistentHashRing in consistent-hashing/consistent_hashing.py

**Date:** 2026-05-29
**Time:** 13:12

# `ConsistentHashRing`

## Purpose

This is a reference implementation of the consistent hashing technique described in DDIA Chapter 6 (Partitioning). It solves the problem of distributing keys across a dynamic set of nodes such that adding or removing a node only reassigns a small fraction of keys — unlike naive modulo hashing, which reshuffles nearly everything.

The ring maps both keys and nodes into a shared 32-bit hash space (0 to 2^32 - 1). Each physical node gets multiple **virtual nodes** (vnodes) placed around the ring. A key is assigned to the first vnode encountered when walking clockwise from the key's hash position. Virtual nodes exist to smooth out load distribution — without them, a few physical nodes can end up owning disproportionately large arcs.

## Contract

**Preconditions:**
- `num_vnodes > 0` and `replication_factor >= 1` (not enforced — the code will silently misbehave with 0 or negative values).
- Node IDs must be unique strings. Weights must be positive floats.

**Postconditions:**
- After `add_node`/`remove_node`, the ring is in a consistent state: `_ring_positions` is sorted, `_ring_nodes` is parallel to it, and `_nodes` reflects the current membership.
- `get_node(key)` is deterministic for a given ring state.

**Invariants:**
- `len(_ring_positions) == len(_ring_nodes)` always.
- `_ring_positions` is always sorted in ascending order.
- Every entry in `_ring_nodes` corresponds to a key in `_nodes` (no orphaned vnodes).

## Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `num_vnodes` | `int` | Base number of virtual nodes per physical node (before weight scaling). Default 150 is a common choice — enough for reasonable balance with 10s of nodes. |
| `replication_factor` | `int` | How many distinct physical nodes `get_nodes()` returns for a key. `get_node()` ignores this and always returns 1. |

## Return Value

The class itself is a stateful container. Key return semantics per method:

- **`add_node` / `remove_node`** → `Dict[Tuple[int,int], Tuple[str,str]]`: a transfer map. Each entry `(arc_start, arc_end) → (from_node, to_node)` tells the caller which key ranges moved between which nodes. This is the data the caller needs to orchestrate actual data migration.
- **`get_node`** → `str`: the single primary owner of a key.
- **`get_nodes`** → `List[str]`: the preference list — RF distinct physical nodes responsible for a key, ordered by ring proximity. This is what Dynamo-style systems use for replication.
- **`get_load_distribution`** → `Dict[str, float]`: fraction of the ring's address space owned by each node. Values sum to 1.0.

## Algorithm

### Ring placement (`add_node`)

For each of `num_vnodes * weight` virtual nodes:
1. Hash `"{node_id}:{i}"` to get a 32-bit position.
2. Use `bisect_left` to find where this position falls in the sorted position list.
3. Before inserting, look at who currently owns that position (the successor vnode at index `idx % len`). Record the arc `[prev_vnode+1, pos]` as a transfer from that old owner to the new node.
4. Insert the position and node ID at `idx` to maintain sort order.

The key insight: because we insert one vnode at a time and compute transfers *before* inserting, each transfer is computed against the ring state that includes all previously inserted vnodes of the same node. This means the transfer map may contain overlapping or superseded arcs — it's an approximation useful for orchestration, not a precise partition map.

### Key lookup (`get_node`)

1. Hash the key to a 32-bit position.
2. `bisect` (right) finds the first ring position strictly greater than the key's hash.
3. If that index is past the end, wrap to index 0 (the ring wraps around).
4. Return the node at that index.

This is the classic "walk clockwise to the first vnode" operation.

### Preference list (`get_nodes`)

Same starting point as `get_node`, but walks clockwise collecting distinct physical nodes until it has `replication_factor` of them. The `seen` set skips over vnodes belonging to already-collected physical nodes — this ensures replicas land on different machines.

### Load distribution (`get_load_distribution`)

Walks every adjacent pair of vnodes on the ring, computes the arc length between them (handling wraparound), and credits it to the node that owns the clockwise vnode. This gives the theoretical fraction of keys each node would own under a uniform key distribution.

## Side Effects

All mutations are internal state changes to `_ring_positions`, `_ring_nodes`, and `_nodes`. No I/O, no external calls. The class is not thread-safe — concurrent `add_node`/`remove_node` calls will corrupt the sorted lists.

## Error Handling

- `remove_node` raises `ValueError` if the node isn't present.
- `get_node` and `get_nodes` raise `ValueError` on an empty ring.
- `get_nodes` raises `ValueError` if there aren't enough physical nodes to satisfy the replication factor.
- `add_node` silently returns `{}` if the node already exists (idempotent).
- No validation on `num_vnodes`, `weight`, or `replication_factor` being positive.

## Usage Patterns

```python
ring = ConsistentHashRing(num_vnodes=150, replication_factor=3)

# Add nodes — use the transfer map to migrate data
transfers = ring.add_node("node-1", weight=1.0)
transfers = ring.add_node("node-2", weight=2.0)  # gets 2x vnodes

# Route a key
primary = ring.get_node("user:42")
replicas = ring.get_nodes("user:42")  # [primary, replica1, replica2]

# Monitor balance
print(ring.load_imbalance())  # 1.0 = perfect, higher = skewed
```

Caller obligations:
- Must actually move data according to the transfer map returned by `add_node`/`remove_node`. The ring doesn't move data — it only tells you what should move.
- Must ensure `replication_factor <= number of physical nodes` before calling `get_nodes`.

## Dependencies

- `hashlib.md5` — the hash function. MD5 is fine here (not used for security, just distribution). Truncated to 32 bits via `& 0xFFFFFFFF`.
- `bisect` — maintains the sorted ring via binary search insertion and lookup. This is what makes `get_node` O(log V) where V is total vnodes, rather than O(V).
- No external dependencies beyond the standard library.

## Assumptions Not Enforced by Types

1. **Weight is positive.** A weight of 0 produces 0 vnodes (node exists in `_nodes` but has no ring presence). Negative weights would also produce 0 vnodes due to `int()` truncation but the node would still appear in `_nodes`.
2. **No hash collisions.** If two vnodes hash to the same position, `bisect_left` will stack them adjacent, but the transfer calculation assumes positions are unique. Collisions are unlikely in a 2^32 space with hundreds of vnodes but not impossible.
3. **`_ring_positions` uses `list.insert`**, which is O(n) per insertion. With 150 vnodes per node and, say, 100 nodes, that's 15,000 positions — fine. At thousands of nodes this would become a bottleneck; a production system would use a balanced tree.
4. **Transfer map accuracy.** Because vnodes are inserted one at a time, later insertions of the *same* node can split arcs that were recorded in the transfer map earlier in the same `add_node` call. The map is therefore a reasonable but not perfectly minimal description of data movement.

## Topics to Explore

- [file] `consistent-hashing/test_consistent_hashing.py` — Validates load balancing properties, transfer correctness, and edge cases like single-node rings
- [function] `consistent-hashing/consistent_hashing.py:_hash` — The hash function choice (MD5 truncated to 32 bits) directly determines distribution quality; worth understanding why MD5 over alternatives like xxhash
- [general] `vnodes-and-load-balance` — How the number of virtual nodes (150 default) relates to load variance; DDIA discusses the tradeoff between memory overhead and balance
- [general] `dynamo-preference-list` — How `get_nodes` implements Dynamo's preference list concept and how it interacts with sloppy quorums and hinted handoff in practice
- [file] `leaderless-replication/dynamo.py` — Likely consumer of this ring for routing reads/writes in a Dynamo-style replication scheme

## Beliefs

- `consistent-hash-ring-sorted-invariant` — `_ring_positions` is maintained in sorted order at all times via `bisect` insertion; `_ring_nodes` is kept parallel to it
- `consistent-hash-ring-vnode-count-scaled-by-weight` — A node with weight W gets `int(num_vnodes * W)` virtual nodes, so weight=2.0 means twice the ring presence and roughly twice the load
- `consistent-hash-ring-get-nodes-skips-same-physical` — `get_nodes` walks clockwise collecting only distinct physical node IDs, ensuring replicas never land on the same machine
- `consistent-hash-ring-transfer-map-approximate` — The transfer map from `add_node` is computed incrementally per-vnode insertion, so it may contain arcs that are subsequently split by later vnodes of the same node in the same call
- `consistent-hash-ring-lookup-is-olog-v` — Key lookup via `get_node` is O(log V) where V is total virtual nodes, thanks to `bisect` on the sorted position list

