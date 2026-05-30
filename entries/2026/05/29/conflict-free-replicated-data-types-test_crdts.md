# File: conflict-free-replicated-data-types/test_crdts.py

**Date:** 2026-05-29
**Time:** 14:02

I'll work from the test file content you provided plus the imports to explain the code.

---

## Purpose

This is the **comprehensive test suite** for a CRDT (Conflict-Free Replicated Data Type) library. It validates four CRDT implementations — `GCounter`, `PNCounter`, `LWWRegister`, and `ORSet` — plus a `CRDTReplicaGroup` coordination helper. The file covers 20 discrete test scenarios organized to verify both the mathematical properties CRDTs must satisfy (commutativity, associativity, idempotency) and their practical distributed behavior (concurrent updates, partial sync, convergence).

Its role in the project is that of a **specification-as-tests**: reading the test names and assertions tells you exactly what the CRDT contract is, even without reading the implementation.

## Key Components

### CRDT Types Under Test

| Class | What it models | Key operations |
|-------|---------------|----------------|
| `GCounter` | Grow-only counter (non-negative) | `increment(n)`, `value()`, `merge()`, `state()` |
| `PNCounter` | Counter supporting both increment and decrement | `increment(n)`, `decrement(n)`, `value()`, `merge()`, `state()` |
| `LWWRegister` | Last-Writer-Wins register (single value) | `set(val, timestamp=)`, `get()`, `get_timestamp()`, `merge()`, `state()` |
| `ORSet` | Observed-Remove set (add-wins semantics) | `add(elem)`, `remove(elem)`, `contains(elem)`, `elements()`, `merge()`, `state()` |

### `CRDTReplicaGroup`

A test harness that manages multiple replicas and provides:
- `get_replica(id)` — retrieve a specific replica by ID
- `sync_all()` — merge every pair so all replicas converge
- `sync(a, b)` — directed merge from replica `a` into `b`
- `all_converged()` — checks whether all replicas hold identical state

This is infrastructure for simulating a distributed system in a single process.

### Test Classes (20 scenarios)

The tests are numbered 1–20 in comments and grouped into classes by concern:

| # | Class | Tests |
|---|-------|-------|
| 1 | `TestGCounterBasic` | Increment accumulates; 3-node sync converges to sum |
| 2 | `TestGCounterProperties` | Merge is commutative, associative, idempotent |
| 3 | `TestGCounterConcurrent` | 60 concurrent increments across 3 replicas converge |
| 4 | `TestPNCounterBasic` | Increment/decrement arithmetic; negative values allowed |
| 5 | `TestPNCounterMerge` | Concurrent inc+dec merges correctly |
| 6 | `TestPNCounterConvergence` | Mixed ops across 3 nodes converge to `4` |
| 7 | `TestLWWRegisterTimestamp` | Higher timestamp wins after sync |
| 8 | `TestLWWRegisterTiebreak` | Same timestamp → deterministic tiebreak by replica ID (lexicographic, higher wins) |
| 9 | `TestLWWRegisterSequential` | Sequential sets auto-increment timestamp to `3.0` |
| 10 | `TestLWWRegisterConcurrent` | 3-node concurrent writes, highest-timestamp value wins |
| 11 | `TestORSetBasic` | Add, remove, contains, elements; remove of absent returns `False` |
| 12 | `TestORSetAddWins` | Concurrent add beats concurrent remove (add-wins semantics) |
| 13 | `TestORSetRemoveKnownTags` | Remove only tombstones locally-known tags; foreign tags survive |
| 14 | `TestORSetAddRemoveAdd` | Re-adding after remove produces a fresh tag; element is present |
| 15 | `TestORSetDisjoint` | Merging disjoint sets unions them |
| 16 | `TestORSetProperties` | Commutativity, associativity, idempotency for OR-Set |
| 17 | `TestReplicaGroupConvergence` | `sync_all` converges for both GCounter and PNCounter |
| 18 | `TestReplicaGroupPartialSync` | Partial sync propagates incrementally; full sync finishes the job |
| 19 | `TestReplicaGroupManyReplicas` | 15 replicas each incrementing; converges to `sum(1..15) = 120` |
| 20 | `TestSerialization` | `state()` returns inspectable dicts for all four CRDT types; LWW supports any JSON-serializable value |

## Patterns

**1. `deepcopy` for merge isolation.** Every merge test copies operands before merging to prove that merge doesn't corrupt the source. This is critical — without it, you couldn't distinguish a correct merge from one that mutates its argument and happens to look right.

**2. Mathematical property testing.** Tests 2 and 16 verify the three semilattice axioms (commutativity, associativity, idempotency) that *define* a valid state-based CRDT. These are the properties that guarantee convergence without coordination. The test structure is formulaic: create distinct states, merge in both orders (or self-merge), assert equality.

**3. `CRDTReplicaGroup` as a network simulator.** Rather than testing merge in isolation, many tests use the group abstraction to simulate multi-node systems. This separates the "network topology" concern from "CRDT correctness" — `sync_all()` is a full-mesh gossip round, `sync(a, b)` is a single directed message.

**4. `__eq__` protocol for convergence checks.** `all_converged()` and the property tests rely on `__eq__` being implemented on each CRDT. The tests implicitly validate that `__eq__` compares semantic state, not object identity.

**5. One class per scenario.** Each numbered scenario gets its own class, making it easy to run a single scenario in isolation (`pytest -k TestORSetAddWins`).

## Dependencies

**Imports:**
- `pytest` — test framework
- `copy.deepcopy` — isolation of CRDT state for non-destructive merge testing
- `crdts` — the implementation module (`GCounter`, `PNCounter`, `LWWRegister`, `ORSet`, `CRDTReplicaGroup`)

**Imported by:**
- `tester_test_crdts.py` — likely a meta-test or test runner harness that validates the test suite itself (consistent with the pattern in other modules like `bloom-filter/tester_test_bloom_filter.py`)

## Flow

Each test follows the same pattern:

1. **Construct** — create replicas (directly or via `CRDTReplicaGroup`)
2. **Mutate** — apply operations (`increment`, `set`, `add`, `remove`) to simulate local writes at different nodes
3. **Sync** — merge state via `merge()` or `sync_all()` / `sync(a, b)`
4. **Assert** — check `value()`, `get()`, `elements()`, `contains()`, `state()`, `all_converged()`, or `__eq__`

For the mathematical property tests (2, 16), the flow is:
1. Create instances with distinct states
2. Merge in different orders (A·B vs B·A) or groupings ((A·B)·C vs A·(B·C))
3. Assert the results are equal

## Invariants

- **Convergence guarantee**: After `sync_all()`, `all_converged()` must return `True` — this is the fundamental CRDT promise.
- **Merge is a semilattice join**: commutative, associative, idempotent. Tests 2 and 16 enforce this.
- **Add-wins semantics for ORSet**: A concurrent add and remove of the same element must result in the element being present (test 12). Remove only affects tags the remover has observed (test 13).
- **LWW deterministic tiebreak**: When timestamps are equal, the higher replica ID wins (test 8 asserts `"from_B"` since `"B" > "A"`).
- **LWW auto-incrementing timestamps**: Sequential `set()` calls without explicit timestamps produce monotonically increasing timestamps (test 9 checks `get_timestamp() == 3.0` after three sets).
- **PNCounter supports negative values**: Decrementing below zero is valid (test 4).
- **`state()` returns plain dicts**: Serialization format is documented by assertion — `GCounter` uses `{"counts": {...}}`, `PNCounter` nests `p` and `n` sub-counters, etc.

## Error Handling

The test suite doesn't test error paths extensively. The only negative-path check is `ORSet.remove("cherry")` returning `False` for a non-existent element (test 11). There are no tests for invalid inputs (e.g., negative increments, non-string replica IDs). The CRDTs appear to be designed for correctness under valid usage rather than defensive input validation.

---

## Topics to Explore

- [file] `conflict-free-replicated-data-types/crdts.py` — The implementation behind all four CRDTs; particularly how OR-Set tracks unique tags per element to achieve add-wins semantics, and how `CRDTReplicaGroup.sync_all()` achieves full-mesh convergence
- [function] `conflict-free-replicated-data-types/crdts.py:ORSet.merge` — The most complex merge logic; understanding how it unions tags and resolves concurrent add/remove is key to understanding tests 12 and 13
- [function] `conflict-free-replicated-data-types/crdts.py:LWWRegister.merge` — How tiebreaking by replica ID is implemented when timestamps match (test 8 depends on this)
- [general] `crdt-semilattice-properties` — The mathematical foundation: why commutativity + associativity + idempotency of merge guarantees eventual consistency without coordination (Chapter 5/9 of DDIA)
- [file] `conflict-free-replicated-data-types/tester_test_crdts.py` — The meta-test layer; understanding what it validates about the test suite itself

## Beliefs

- `crdt-tests-cover-20-scenarios` — The test suite is organized into exactly 20 numbered scenarios, each in its own test class, covering all four CRDT types plus the replica group harness
- `orset-add-wins-semantics` — OR-Set uses add-wins conflict resolution: a concurrent add and remove of the same element always resolves to the element being present, because remove only tombstones locally-observed tags
- `lww-tiebreak-by-replica-id` — When two LWW-Register writes have identical timestamps, the lexicographically higher replica ID wins deterministically
- `pncounter-composed-of-two-gcounters` — PNCounter state is structured as two sub-counters (`p` for increments, `n` for decrements), confirming the standard "positive minus negative" decomposition
- `crdt-merge-returns-self` — `merge()` mutates and returns `self` (enabling chaining like `deepcopy(a).merge(b)`), rather than returning a new instance

