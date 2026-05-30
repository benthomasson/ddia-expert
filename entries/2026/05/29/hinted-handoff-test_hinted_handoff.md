# File: hinted-handoff/test_hinted_handoff.py

**Date:** 2026-05-29
**Time:** 13:10

I'll work from the test file content provided, which gives a thorough view of the implementation's contract.

---

# `hinted-handoff/test_hinted_handoff.py`

## Purpose

This is the test suite for a **hinted handoff** implementation — a technique from Dynamo-style distributed databases (DDIA Chapter 5). Hinted handoff ensures write availability when some replica nodes are temporarily down: writes intended for unavailable nodes are temporarily stored as "hints" on substitute nodes, then delivered ("handed off") when the target node recovers.

This file validates the full lifecycle: normal replication, sloppy quorum fallback, hint creation, hint delivery on recovery, hint expiration, and failure edge cases.

## Key Components

### `make_store(**kwargs)` (helper factory)

Constructs a `HintedHandoffStore` with sensible defaults: 5 nodes (`A`–`E`), replication factor 3, read/write quorum 2, hint TTL of 50 time units, and sloppy quorum enabled. Tests override individual parameters (e.g., `sloppy_quorum=False`, `hint_ttl=10`) to isolate specific behaviors.

### Imported types from `hinted_handoff`

- **`HintedHandoffStore`** — The coordinator. Owns the ring of nodes, routes `put`/`get` operations, manages quorum logic, and orchestrates handoff. Key methods exercised: `put()`, `get()`, `get_preferred_nodes()`, `set_node_available()`, `trigger_handoff()`, `expire_all_hints()`.
- **`Node`** — A single replica. Stores key-value data and a list of hints destined for other nodes. Exposes `get()`, `hint_count()`, and a `hints` attribute (list, directly clearable).
- **`Hint`** — A data class representing a deferred write: carries the target node, key, value, version, and creation timestamp.

## Patterns

**Logical clock injection**: Every mutating operation takes a `current_time` parameter instead of reading wall-clock time. This makes all timing behavior (TTL expiration, version ordering) fully deterministic and testable.

**Result-dict protocol**: `put()` and `get()` return plain dicts with well-defined keys (`success`, `sloppy`, `hints_stored`, `replicas_written`, `value`, `version`). Tests assert on these structured results rather than side effects, making the contract explicit.

**Preference-list testing**: Tests call `get_preferred_nodes()` first to discover which nodes *should* own a key, then selectively disable those nodes. This avoids hardcoding node assignments that depend on the hashing function.

**Progressive failure simulation**: Tests build up from a single node failure (tests 2–5) through two failures (tests 7–8) to all-preferred-down (test 11) and hint-node-crash (test 12), systematically covering the failure matrix.

## Dependencies

**Imports**: `pytest` (test framework), `Hint`, `Node`, `HintedHandoffStore` from `hinted_handoff.py`.

**Imported by**: Nothing — this is a leaf test module. The companion `tester_test_hinted_handoff.py` likely wraps or validates these tests as part of a meta-testing framework used across the project.

## Flow

The tests follow a consistent sequence:

1. **Setup** — `make_store()` creates a fresh cluster.
2. **Discover topology** — `get_preferred_nodes(key)` returns the N nodes responsible for a key (consistent hashing order).
3. **Inject failure** — `set_node_available(node_id, False)` marks nodes as down.
4. **Write** — `put(key, value, current_time=t)` attempts a quorum write. If preferred nodes are down and sloppy quorum is enabled, substitute nodes accept the write and store hints.
5. **Recover** — `set_node_available(node_id, True)` brings the node back.
6. **Handoff** — `trigger_handoff(node_id, current_time=t)` scans all nodes for hints targeting the recovered node, delivers the data, and removes the hints.
7. **Verify** — Assert the recovered node has the correct data, hints are cleared, and reads return the latest version.

## Invariants

1. **Quorum arithmetic**: A write succeeds iff at least `write_quorum` nodes (preferred or substitute) accept it. With `sloppy_quorum=False`, only preferred nodes count.
2. **Hint targeting**: Every hint records which *preferred* node was unavailable (`target_node`) and which *substitute* node is holding it (`hint_node`). The hint node is always outside the preferred list.
3. **Version monotonicity**: Successive `put()` calls with increasing `current_time` produce increasing versions. After handoff, the recovered node holds the latest version, not an earlier one (test 9).
4. **Hint cleanup**: After successful handoff delivery, hint count on the substitute node drops to zero (test 4).
5. **TTL semantics**: A hint created at time `t` with TTL `d` expires at time `t + d` (exclusive — test 5 shows `t=1, ttl=10` survives at `t=5`, expires at `t=11`).
6. **Deterministic key mapping**: `get_preferred_nodes(key)` always returns the same list for the same key (test 10).

## Error Handling

The implementation uses **result dicts** rather than exceptions. A failed write returns `{"success": False, "sloppy": False}` (test 6) — no exception is raised. This is consistent with Dynamo-style systems where unavailability is an expected operational state, not an exceptional condition.

Test 12 is notable: it simulates a hint node crash by clearing `node.hints` directly, showing that hint loss is a silent data-loss scenario — the recovered node simply never receives the data. The system doesn't detect or recover from this; it's an acknowledged limitation of hinted handoff (which is why Dynamo pairs it with anti-entropy via Merkle trees).

---

## Topics to Explore

- [file] `hinted-handoff/hinted_handoff.py` — The implementation: how consistent hashing selects preferred nodes, how sloppy quorum routing works, and how `trigger_handoff` iterates nodes to deliver hints
- [function] `hinted-handoff/hinted_handoff.py:get_preferred_nodes` — The consistent hashing logic that determines which N nodes own a key — foundational to understanding every test's setup
- [file] `read-repair/read_repair.py` — Read repair is the complementary consistency mechanism: hinted handoff handles write-time failures, read repair catches divergence at read time
- [file] `merkle-tree/merkle_tree.py` — Merkle trees provide anti-entropy repair for the exact scenario test 12 exposes: when hints are lost, background Merkle tree comparison detects and repairs missing data
- [general] `sloppy-vs-strict-quorum` — DDIA Chapter 5's distinction between strict quorums (W + R > N on the *same* N nodes) and sloppy quorums (any W nodes will do) — tests 6 and 7 directly contrast these two modes

## Beliefs

- `hinted-handoff-sloppy-quorum-writes-to-non-preferred` — When sloppy quorum is enabled and preferred replicas are down, writes succeed by routing to non-preferred substitute nodes that store hints for later delivery
- `hinted-handoff-hint-ttl-exclusive-boundary` — Hint expiration uses exclusive comparison: a hint created at time `t` with TTL `d` is not expired at time `t + d - 1` but is expired at time `t + d`
- `hinted-handoff-strict-quorum-rejects-insufficient-preferred` — With `sloppy_quorum=False`, a write fails if fewer than `write_quorum` preferred nodes are available, even if non-preferred nodes could serve
- `hinted-handoff-hints-cleared-after-delivery` — After `trigger_handoff` successfully delivers hints to a recovered node, the hint count on the substitute node drops to zero
- `hinted-handoff-hint-loss-is-silent-data-loss` — If a hint node crashes and its hints are lost before handoff, the target node never receives the data and no error is raised — this is an acknowledged limitation requiring anti-entropy repair

