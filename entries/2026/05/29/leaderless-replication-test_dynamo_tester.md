# File: leaderless-replication/test_dynamo_tester.py

**Date:** 2026-05-29
**Time:** 10:22

## Purpose

`test_dynamo_tester.py` is a **QA-level integration test suite** for the Dynamo-style leaderless replication module. It sits alongside the developer-written `test_dynamo.py` and serves as an independent validation layer — the "tester" naming convention across this repo (`tester_test_*.py` or `test_*_tester.py`) indicates tests generated or curated to verify the implementation against its spec, distinct from the author's own unit tests.

This file exercises the full public API of `dynamo.py`: cluster formation, quorum reads/writes, node failure handling, read repair, conflict detection, sloppy quorums with hinted handoff, and anti-entropy repair.

## Key Components

The file defines **10 test functions**, each targeting a specific behavioral contract:

| Test | What it validates |
|------|-------------------|
| `test_spec_example_full` | End-to-end happy path: write, read, node failure, quorum enforcement, read repair, anti-entropy |
| `test_spec_sloppy_quorum_example` | Sloppy quorum stores hints on live nodes and delivers them when the target recovers |
| `test_read_nonexistent_key` | Reading a missing key returns `None`/version 0 without conflict |
| `test_version_rollback_on_failed_write` | A failed quorum write must not increment the version counter — the next successful write picks up from the last committed version |
| `test_node_unavailable_rejects_operations` | `ReplicaNode` rejects reads/writes when marked unavailable, preserving prior state |
| `test_multiple_keys_independent` | Version counters are per-key, not global |
| `test_conflict_detection` | When replicas hold the same version number but different values, a read returns `is_conflict=True` with all conflicting values as a list |
| `test_large_cluster_10k_ops` | N=7 cluster handles 10k writes across 50 keys without error (smoke/perf test) |
| `test_anti_entropy_full_sync` | Anti-entropy repair synchronizes a node that missed all writes while offline |
| `test_no_hints_without_sloppy_quorum` | With `sloppy_quorum=False`, no hints are stored or delivered — offline nodes stay empty until anti-entropy runs |

## Patterns

**Failure injection via `set_node_available`**: Every test that exercises fault tolerance follows the same pattern — take nodes down, perform operations, bring nodes back, assert repair behavior. This is a controlled simulation of network partitions.

**Direct store manipulation for conflict injection** (test 7): Rather than orchestrating a real concurrent-write scenario, the test injects divergent `VersionedValue` entries directly into `_store` on each node. This is a pragmatic shortcut — it tests conflict *detection* without requiring the implementation to produce conflicts organically.

**Quorum arithmetic as a test parameter**: Tests carefully choose `write_quorum` and `read_quorum` values to create the exact failure conditions needed. For instance, `test_version_rollback_on_failed_write` uses `W=3, R=1` so that a single node failure guarantees write failure.

**Naming convention**: The file is `test_dynamo_tester.py` (not `tester_test_dynamo.py` like most other modules), which is a minor inconsistency in the repo's naming scheme.

## Dependencies

**Imports from `dynamo.py`:**
- `DynamoCluster` — the cluster facade (primary entry point)
- `QuorumNotMet` — exception raised when insufficient replicas respond
- `ReplicaNode` — individual storage node (tested directly in test 5)
- `VersionedValue` — data class for value + version + origin node (used to inject conflicts in test 7)
- `ReadResult` — return type of `cluster.get()` with `.value`, `.version`, `.is_conflict`, `.replicas_repaired`

**Nothing imports this file** — it's a leaf test module.

## Flow

Tests follow a consistent structure:

1. **Construct** a `DynamoCluster` with specific quorum parameters
2. **Setup** — write initial data, optionally take nodes offline
3. **Act** — perform the operation under test (put, get, repair, deliver hints)
4. **Assert** — verify return values, side effects on specific nodes, or exception behavior

Test 1 (`test_spec_example_full`) is the most complex, walking through an entire lifecycle: write → read → node failure → write with degraded quorum → node recovery → read repair → quorum failure → anti-entropy.

## Invariants

These tests collectively enforce:

- **Quorum semantics**: A write requires exactly `W` successful replica writes; fewer raises `QuorumNotMet`. Reads require `R` responses.
- **Per-key versioning**: Version numbers are scoped to each key. Writing to key `"a"` twice gives version 2; key `"b"` written once stays at version 1.
- **Version monotonicity without gaps**: Failed writes must not consume version numbers — the version counter only advances on successful quorum writes.
- **Read repair is transparent**: When a read discovers stale replicas, it updates them and reports `replicas_repaired >= 1` without requiring explicit action.
- **Conflict = same version, different values**: Two replicas with the same version but different values trigger `is_conflict=True` and return all conflicting values as a list.
- **Sloppy quorum is opt-in**: Hints are only stored when `sloppy_quorum=True`. Without it, offline nodes receive nothing until anti-entropy runs.

## Error Handling

- `QuorumNotMet` is the only exception tested. It is expected (via `pytest.raises`) when too many nodes are down to satisfy the write quorum.
- Unavailable nodes return `False` from `write()` and `None` from `read()` — failures are values, not exceptions, at the node level. The cluster layer aggregates these into quorum decisions.
- There is no test for partial read quorum failure (e.g., R=2 but only 1 node responds) — this is a coverage gap.

## Topics to Explore

- [file] `leaderless-replication/dynamo.py` — The implementation under test; understanding `DynamoCluster.put`/`get` internals explains why the quorum arithmetic in these tests works
- [function] `leaderless-replication/dynamo.py:anti_entropy_repair` — How the repair mechanism discovers and resolves inconsistencies across replicas
- [file] `leaderless-replication/test_dynamo.py` — The developer-written tests; comparing coverage with this tester file reveals what each suite uniquely validates
- [general] `quorum-intersection-guarantee` — Why R + W > N ensures at least one node in a read set has the latest write — the theoretical foundation for every quorum parameter choice in these tests
- [general] `sloppy-quorum-tradeoffs` — How sloppy quorums trade consistency for availability, and why hinted handoff is necessary to eventually converge

## Beliefs

- `dynamo-version-rollback-on-failed-write` — A failed quorum write must not increment the version counter; the next successful write uses the version that would have been assigned to the failed one
- `dynamo-conflict-detection-same-version-different-values` — When multiple replicas hold different values at the same version number, `ReadResult.is_conflict` is `True` and `value` is a list of all conflicting values
- `dynamo-sloppy-quorum-opt-in` — Hinted handoff only occurs when `DynamoCluster` is constructed with `sloppy_quorum=True`; without it, `deliver_hints()` returns 0 and offline nodes stay empty
- `dynamo-per-key-versioning` — Version counters are scoped per key, not global across the cluster; independent keys maintain independent version sequences
- `dynamo-read-repair-on-get` — A `cluster.get()` call that discovers stale replicas automatically repairs them and reports the count via `ReadResult.replicas_repaired`

