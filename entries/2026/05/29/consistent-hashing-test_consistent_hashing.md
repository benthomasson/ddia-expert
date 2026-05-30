# File: consistent-hashing/test_consistent_hashing.py

**Date:** 2026-05-29
**Time:** 12:54

I'll work from the test file contents, which reveal the implementation's full API surface.

---

# `consistent-hashing/test_consistent_hashing.py`

## Purpose

This is the test suite for a consistent hashing ring implementation — one of the DDIA reference implementations. It validates not just correctness but the **distributed-systems properties** that make consistent hashing useful: determinism, minimal disruption on membership changes, load balance, and replication placement. The test file effectively serves as a specification for the `ConsistentHashRing` class.

## Key Components

The file is a flat collection of pytest test functions (no classes). Each test targets a specific property of the ring:

| Test | What it validates |
|------|------------------|
| `test_determinism` | Same key always maps to the same node — the fundamental contract |
| `test_basic_routing` | A key resolves to one of the added nodes |
| `test_replication_distinct` | `get_nodes()` returns `replication_factor` **distinct physical nodes**, and the first matches `get_node()` |
| `test_replication_insufficient_nodes` | Raises `ValueError` when fewer physical nodes exist than `replication_factor` |
| `test_load_balance` | With 150 vnodes and 3 nodes, imbalance stays below 1.5× |
| `test_load_distribution_sums_to_one` | Load fractions are a proper probability distribution |
| `test_minimal_redistribution` | Adding a 4th node moves roughly 1/N of keys — not a full reshuffle |
| `test_node_removal` | Removing a node drops it cleanly; remaining keys still resolve |
| `test_weighted_nodes` | A node with weight 3× gets roughly 3× the keys |
| `test_empty_ring` | `get_node()` on an empty ring raises `ValueError` |
| `test_single_node` | Degenerate case: one node gets 100% of traffic |
| `test_duplicate_add_idempotent` | Adding the same node twice doesn't create duplicates |
| `test_ring_position_valid_range` | Hash positions fit in `[0, 2³²)` — a 32-bit hash ring |
| `test_scalability` | 100 nodes × 150 vnodes (15,000 ring positions) still works |
| `test_ring_info` | Human-readable summary string includes node count and RF |
| `test_add_node_returns_transfers` | `add_node()` returns a transfer map showing which key ranges moved and between which nodes |
| `test_remove_node_returns_transfers` | `remove_node()` likewise returns a transfer map |
| `test_vnode_count_affects_balance` | More vnodes → lower imbalance, monotonically at the extremes |

## Patterns

**Property-based testing (manual).** Rather than checking exact hash outputs, the tests assert statistical properties (`moved < 4000`, `ratio` between 2.0 and 4.5, `imbalance < 1.5`). This makes tests resilient to hash function changes while still catching real regressions.

**Boundary-value coverage.** The suite explicitly tests empty ring, single node, duplicate add, and insufficient-nodes-for-replication — all the edge cases a naïve implementation would get wrong.

**Transfer-map assertions.** `test_add_node_returns_transfers` and `test_remove_node_returns_transfers` verify that membership changes return structured data about which key ranges migrated between which nodes. This is an operational concern — in production, you'd use this to orchestrate data movement.

## Dependencies

- **Imports:** `pytest` (test framework), `ConsistentHashRing` from `consistent_hashing` (the module under test).
- **Imported by:** Nothing imports this file. The companion `tester_test_consistent_hashing.py` likely validates the test suite itself (a "tester-of-the-tests" pattern seen across this repo).

## Flow

Each test follows the same pattern:
1. Construct a `ConsistentHashRing` with specific `num_vnodes` and optionally `replication_factor`.
2. Add nodes (optionally with `weight`).
3. Assert properties of key lookups, load distribution, or membership operations.

The statistical tests (`test_minimal_redistribution`, `test_weighted_nodes`, `test_load_balance`) use large key populations (10,000 keys) to get stable distributions, then assert bounds loose enough to tolerate hash variance but tight enough to catch broken logic.

`test_vnode_count_affects_balance` sweeps across vnode counts `[1, 10, 50, 150, 500]` and asserts the general trend: the last (500 vnodes) produces better balance than the first (1 vnode). It doesn't require strict monotonicity across all entries — just that the extremes are ordered.

## Invariants

1. **Determinism:** `get_node(key)` is a pure function of ring state — no randomness.
2. **Hash range:** Ring positions are in `[0, 2³²)`, meaning the ring uses a 32-bit hash space (likely MD5 truncated, SHA-256 truncated, or CRC32).
3. **Replication safety:** `get_nodes()` must return distinct physical nodes (not just distinct vnodes). If fewer physical nodes exist than `replication_factor`, it raises rather than silently under-replicating.
4. **Consistency between single and multi-node lookup:** `get_nodes(key)[0] == get_node(key)` — the primary replica is always the first in the preference list.
5. **Idempotent add:** Adding a node that already exists doesn't change the ring.
6. **Minimal disruption:** Adding an Nth node moves approximately `1/N` of keys, not more.
7. **Transfer maps:** Both `add_node()` and `remove_node()` return `dict[(start, end), (from_node, to_node)]` describing range ownership changes.

## Error Handling

The test suite validates two error conditions:
- **Empty ring:** `get_node()` raises `ValueError` — the ring refuses to route when no nodes exist.
- **Insufficient nodes for replication:** `get_nodes()` raises `ValueError` when `replication_factor > len(physical_nodes)` — it will not silently degrade replication guarantees.

Both use `pytest.raises(ValueError)`, meaning the implementation uses standard Python exceptions rather than custom error types.

---

## Topics to Explore

- [file] `consistent-hashing/consistent_hashing.py` — The ring implementation: how vnodes are placed, how the bisect-based lookup works, and how weighted nodes multiply vnode count
- [function] `consistent-hashing/consistent_hashing.py:add_node` — How the transfer map is computed during node addition — this is the key operational primitive
- [general] `vnode-balance-tradeoffs` — Why 150 vnodes is the default: the tradeoff between memory/lookup cost and load uniformity, and how this interacts with weighted nodes
- [file] `consistent-hashing/tester_test_consistent_hashing.py` — The "tester of tests" pattern used across this repo — what meta-properties does it check about the test suite itself?
- [general] `consistent-hashing-replication-preference-list` — How `get_nodes()` walks the ring skipping duplicate physical nodes to build a Dynamo-style preference list

## Beliefs

- `consistent-hash-ring-uses-32-bit-space` — The ring's hash space is `[0, 2³²)`, as validated by `test_ring_position_valid_range`
- `consistent-hash-get-nodes-first-equals-get-node` — `get_nodes(key)[0]` always equals `get_node(key)` — the preference list's head is the primary replica
- `consistent-hash-add-node-is-idempotent` — Adding a node that already exists does not change the ring's node count or key assignments
- `consistent-hash-replication-requires-sufficient-physical-nodes` — `get_nodes()` raises `ValueError` rather than silently under-replicating when `replication_factor` exceeds the number of physical nodes
- `consistent-hash-membership-change-returns-transfer-map` — Both `add_node()` and `remove_node()` return a `dict[(start, end), (from_node, to_node)]` describing which key ranges moved between which nodes

