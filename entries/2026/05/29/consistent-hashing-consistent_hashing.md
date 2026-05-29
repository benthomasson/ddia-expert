# File: consistent-hashing/consistent_hashing.py

**Date:** 2026-05-29
**Time:** 08:53

# `consistent-hashing/consistent_hashing.py`

## Purpose

This file implements a **consistent hash ring** — the core data partitioning primitive described in DDIA Chapter 6 (Partitioning). It owns the mapping from arbitrary string keys to physical nodes, handling the three main concerns of consistent hashing: virtual nodes for load balancing, weighted nodes for heterogeneous hardware, and replication via preference lists.

This is the kind of component that sits inside a distributed storage coordinator (like Dynamo, Cassandra, or Riak) to decide which node owns which key range.

## Key Components

### Constants

- **`RING_SIZE = 2**32`** — The hash ring is a 32-bit address space (0 to 4,294,967,295). All positions wrap modulo this value.

### `_hash(key: str) -> int`

A free function that hashes strings to ring positions using MD5, masked to 32 bits. MD5 is not cryptographically relevant here — what matters is uniform distribution across the ring. The `& 0xFFFFFFFF` mask ensures the output fits within `RING_SIZE`.

### `ConsistentHashRing`

The main class. Its internal state is three parallel structures:

| Field | Type | Role |
|-------|------|------|
| `_ring_positions` | `List[int]` | Sorted list of virtual node hash positions |
| `_ring_nodes` | `List[str]` | Node ID at each position (parallel to `_ring_positions`) |
| `_nodes` | `Dict[str, float]` | Physical node registry: node ID → weight |

**Constructor parameters:**

- `num_vnodes` (default 150) — base virtual node count per physical node. Multiplied by weight.
- `replication_factor` (default 1) — how many distinct physical nodes a key maps to.

#### Core Operations

**`add_node(node_id, weight=1.0) -> Dict`** — Inserts a physical node into the ring by generating `int(num_vnodes * weight)` virtual nodes. Each virtual node's position is `_hash(f"{node_id}:{i}")`. For each inserted position, it computes the transfer: which arc of the ring is being taken from which existing node. Returns a transfer map of `{(arc_start, arc_end): (old_owner, new_owner)}`.

**`remove_node(node_id) -> Dict`** — Removes all virtual nodes for the given physical node. Iterates indices in reverse to avoid index invalidation during deletion. For each removed position, finds the successor node that inherits the arc. Returns the same transfer map format.

**`get_node(key) -> str`** — The primary lookup. Hashes the key, then uses `bisect.bisect` to find the first ring position ≥ the hash. If the hash exceeds all positions, it wraps to index 0. This is the classic clockwise-walk on the ring.

**`get_nodes(key) -> List[str]`** — Builds a **preference list** of `replication_factor` distinct physical nodes by walking clockwise from the key's position, skipping duplicate physical nodes (since consecutive virtual nodes might belong to the same physical node). This is exactly how Dynamo constructs its coordinator list.

#### Analytical Methods

- **`get_load_distribution()`** — Computes the fraction of the ring each node owns by summing arc lengths between consecutive virtual nodes. Handles the wrap-around case where `cur_pos < prev_pos`.
- **`get_key_distribution(keys)`** — Empirical distribution: counts how many of the given keys route to each node.
- **`load_imbalance()`** — `max_load / average_load`. Perfect balance is 1.0; higher values indicate skew.
- **`ring_info()`** — Human-readable summary showing node count, weights, vnodes, and load percentages.

## Patterns

1. **Sorted array + bisect** — Rather than a balanced BST or skip list, the ring uses `bisect` on a sorted Python list for O(log n) lookups and O(n) inserts. This is the right tradeoff for a ring that changes rarely (node add/remove) but is queried constantly (key lookups).

2. **Parallel arrays** — `_ring_positions` and `_ring_nodes` are kept in lockstep rather than using a list of tuples. This allows `bisect` to work directly on positions without a custom comparator.

3. **Virtual nodes with deterministic naming** — Virtual node positions are `_hash(f"{node_id}:{i}")`, making them reproducible. The same node added on different machines produces the same ring positions — important for consistency across replicas of the routing table.

4. **Transfer maps** — Both `add_node` and `remove_node` return structured descriptions of what data must move, rather than performing the movement themselves. This separates the partitioning decision from the data migration concern.

## Dependencies

**Imports:**
- `hashlib` — MD5 for position hashing
- `bisect` — Binary search on the sorted ring
- `typing` — Type annotations only (`Tuple` is imported but unused)

**Imported by:**
- `test_consistent_hashing.py` — Unit tests
- `tester_test_consistent_hashing.py` — Likely an automated test validator

No runtime dependencies beyond the standard library.

## Flow

### Adding a node and looking up a key

```
add_node("A", weight=1.0)
  → generates 150 positions via _hash("A:0"), _hash("A:1"), ...
  → for each position, bisect_left finds insertion point
  → records which existing node loses that arc
  → inserts into _ring_positions and _ring_nodes

get_node("user:123")
  → _hash("user:123") → position P
  → bisect(positions, P) → index of first position ≥ P
  → wrap to 0 if past end
  → return _ring_nodes[index]
```

### Preference list construction (`get_nodes`)

```
get_nodes("user:123")
  → find starting index via bisect
  → walk clockwise through _ring_nodes
  → collect nodes into result, skipping duplicates
  → stop when len(result) == replication_factor
```

The walk can traverse up to `len(_ring_positions)` entries in the worst case (when most virtual nodes belong to a small number of physical nodes), but terminates early once enough distinct nodes are found.

## Invariants

1. **`_ring_positions` is always sorted** — enforced by `bisect.insort`-style insertion (`bisect_left` + `insert`).
2. **`_ring_positions` and `_ring_nodes` are always the same length** — every insert/remove touches both.
3. **Each physical node gets `int(num_vnodes * weight)` virtual nodes** — weight < 1.0 reduces presence on the ring.
4. **`get_nodes` always returns exactly `replication_factor` distinct physical nodes** — guaranteed by the precondition check that `len(self._nodes) >= self.replication_factor`.
5. **Adding the same node twice is a no-op** — the early return `if node_id in self._nodes: return {}` prevents duplicate virtual nodes.
6. **Ring positions wrap at `RING_SIZE` (2^32)** — arc computations and the `get_node` wrap-around handle this explicitly.

## Error Handling

Errors are raised eagerly at the API boundary:

- **`get_node` / `get_nodes`** on an empty ring → `ValueError("Empty ring")`
- **`get_nodes`** with insufficient nodes for RF → `ValueError` with a message showing actual vs. required count
- **`remove_node`** for a non-existent node → `ValueError`
- **`add_node`** for an existing node → silent no-op (returns `{}`)

No try/except blocks exist in this module — it raises on invalid state and trusts callers to provide valid inputs.

## Topics to Explore

- [file] `consistent-hashing/test_consistent_hashing.py` — How the ring is tested for load balance, replication correctness, and node add/remove transfer accuracy
- [function] `consistent_hashing.py:get_nodes` — Compare this preference list construction to Dynamo's approach described in DDIA §6; note how virtual nodes complicate the "walk clockwise and collect N nodes" strategy
- [file] `leaderless-replication/dynamo.py` — Likely consumes a consistent hash ring for its partitioning layer; see how the two modules compose
- [general] `virtual-node-count-tuning` — The default of 150 vnodes is a significant design choice — explore how it affects load balance variance vs. memory/computation cost
- [general] `transfer-map-accuracy` — The transfer computation in `add_node` may over-count when multiple virtual nodes for the new node land in the same existing arc; worth verifying against the tests

## Beliefs

- `ch-ring-sorted-invariant` — `_ring_positions` is maintained in sorted order at all times; all lookups depend on this via `bisect`
- `ch-vnode-count-is-weighted` — A node with weight `w` gets `int(num_vnodes * w)` virtual nodes, making weight=0.5 produce half the ring presence
- `ch-preference-list-skips-duplicates` — `get_nodes` walks clockwise past virtual nodes belonging to already-seen physical nodes, ensuring RF distinct physical nodes in the result
- `ch-transfer-maps-are-descriptive` — `add_node` and `remove_node` return transfer descriptions but do not move data themselves; the caller is responsible for acting on the map
- `ch-md5-masked-to-32-bits` — The hash function truncates MD5's 128-bit output to 32 bits via `& 0xFFFFFFFF`, matching the `RING_SIZE` of 2^32

