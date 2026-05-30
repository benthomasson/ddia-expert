# File: byzantine-fault-tolerance/test_pbft.py

**Date:** 2026-05-29
**Time:** 11:53

I can't access the implementation file due to permissions. I'll explain based on the test file itself and the imports, which reveal the full public API surface.

---

## Purpose

`test_pbft.py` is the correctness test suite for a **Practical Byzantine Fault Tolerance (PBFT)** consensus simulation. It validates that the implementation correctly handles the three-phase PBFT protocol (pre-prepare → prepare → commit), tolerates up to `f` Byzantine faults in a `3f+1` node cluster, and recovers via view changes when the primary fails.

This file owns the behavioral contract: it defines what "correct PBFT" means for this codebase.

## Key Components

### Imports from `pbft`

The test imports six symbols that constitute the public API:

| Symbol | Role |
|--------|------|
| `Message` | Protocol message with type, view, sequence number, digest, sender, and optional payload |
| `MessageType` | Enum: `PRE_PREPARE`, `PREPARE`, `COMMIT` (and likely `VIEW_CHANGE`, `NEW_VIEW`) |
| `ByzantineMode` | Enum of fault behaviors: `HONEST`, `SILENT`, `EQUIVOCATING`, `WRONG_DIGEST` |
| `PBFTNode` | Single consensus participant — constructor takes `(node_id, n, f)` |
| `PBFTCluster` | Orchestrator that wires nodes together, delivers messages, and runs the protocol loop |
| `compute_digest` | Deterministic hash of a request string, used for message integrity |

### `PBFTCluster` API (inferred from usage)

- `PBFTCluster(n, f, byzantine_nodes=None)` — Creates an `n`-node cluster tolerating `f` faults. Validates `n >= 3f+1` and `len(byzantine_nodes) <= f`.
- `submit_request(request: str) -> bool` — Full end-to-end: primary proposes, protocol runs to completion, returns success.
- `verify_agreement() -> bool` — Checks that all honest nodes have identical executed logs.
- `get_executed_log() -> list[str]` — Returns the request log from honest nodes.
- `trigger_view_change() -> int` — Forces a view change; returns the new view number.
- `run_protocol(max_rounds)` — Delivers pending messages for a bounded number of rounds.
- `get_node(id) -> PBFTNode` — Direct node access.

### `PBFTNode` API (inferred)

- `receive_message(msg) -> list[Message]` — Processes one message, returns response messages.
- `submit_request(request) -> list[Message]` — Primary-only: creates a `PRE_PREPARE`.
- `is_primary: bool` — Whether this node is the current view's primary.
- `prepared_requests` — Collection of requests that reached the prepared state.
- `byzantine_mode` — The node's fault behavior.
- `_executed_log: list[tuple[int, str]]` — Internal log of `(sequence_number, request)` pairs.

## Patterns

**Progressive fault severity.** Tests are organized from simplest (no faults, test 1) through increasingly adversarial scenarios: silent nodes (test 3), equivocating nodes (test 4), wrong-digest nodes (test 5), view changes (tests 6–7), and larger clusters (test 9). This mirrors the PBFT paper's proof structure.

**Agreement as universal invariant.** Nearly every test calls `cluster.verify_agreement()` — this is the safety property (all honest nodes agree). Test 8 checks it after every single request in a 10-request sequence, making it an explicit invariant test.

**Cluster-as-test-harness.** `PBFTCluster` abstracts away message routing, acting as a deterministic network simulator. Tests never manually deliver messages except in tests 7 and 12, where partial protocol execution is needed to test intermediate states.

**Byzantine injection via constructor.** Faults are configured declaratively at cluster creation with a `{node_id: ByzantineMode}` dict, not by monkey-patching. This keeps tests readable and ensures fault injection is a first-class concept.

## Dependencies

**Imports:**
- `pytest` — test framework
- `pbft` — the implementation module (`Message`, `MessageType`, `ByzantineMode`, `PBFTNode`, `PBFTCluster`, `compute_digest`)

**Imported by:**
- `tester_test_pbft.py` — likely a meta-test or test runner that validates these tests themselves (consistent with the repo's `tester_test_*.py` convention).

## Flow

A typical test follows this pattern:

1. **Construct** a `PBFTCluster` with a fault configuration
2. **Submit** one or more requests via `cluster.submit_request()`
3. **Assert** success (the return value) and agreement (`verify_agreement()`)
4. **Inspect** the executed log (`get_executed_log()`) for ordering correctness

The two exceptions:

- **Test 7** (`test_view_change_repropose_prepared`) manually calls `run_protocol(max_rounds=2)` to stop the protocol mid-stream, inspects the `prepared_requests` state, then forces a view change — testing that prepared-but-uncommitted requests survive leadership transitions.
- **Test 12** (`test_duplicate_message_rejection`) manually constructs a `Message` and calls `receive_message` twice on the same node, verifying that the second delivery produces no output.

## Invariants

1. **`n >= 3f + 1`** — The cluster rejects construction if `n` is too small for the given `f` (test 10 confirms `PBFTCluster(n=5, f=1)` raises `ValueError` since 5 < 3×1+1=4... actually 5 > 4, so this likely enforces `n = 3f + 1` exactly, which matches the standard PBFT formula).
2. **`len(byzantine_nodes) <= f`** — Cannot inject more faults than the tolerance allows (test 10).
3. **Sequence number ordering** — Executed logs maintain monotonically increasing, contiguous sequence numbers starting from 1 (test 13: `seqs == list(range(1, 6))`).
4. **Duplicate rejection** — A node processes each `(view, sequence, digest)` tuple at most once (test 12).
5. **Sender validation** — Messages from node IDs outside `[0, n)` are silently dropped (test 14: sender 99 and -1).
6. **Agreement** — All honest nodes' executed logs are identical at any point (test 8, and most other tests).
7. **Quorum for client acceptance** — At least `f + 1` matching replies are required (test 11).

## Error Handling

- **`ValueError`** on invalid cluster parameters — raised by `PBFTCluster.__init__` when `n` vs `f` is wrong or too many byzantine nodes are specified.
- **Silent drop** for invalid messages — `receive_message` returns an empty list `[]` rather than raising. This is correct for a Byzantine-tolerant protocol: you can't trust the sender, so you just ignore bad input.
- **Boolean success** from `submit_request` — no exceptions for protocol-level failures; the caller checks the return value.

---

## Topics to Explore

- [file] `byzantine-fault-tolerance/pbft.py` — The full PBFT implementation: three-phase protocol state machine, view change logic, and Byzantine fault injection
- [function] `byzantine-fault-tolerance/pbft.py:PBFTCluster.run_protocol` — How message delivery and round progression work in the deterministic simulation
- [function] `byzantine-fault-tolerance/pbft.py:PBFTNode._handle_view_change` — View change is the most complex part of PBFT; understanding how prepared requests are carried across views is critical
- [general] `pbft-quorum-arithmetic` — Why `3f+1` nodes, `2f+1` quorums, and `f+1` reply thresholds: the math behind test 10's validation constraints
- [file] `raft-consensus/test_raft.py` — Compare the Raft consensus test suite against PBFT: Raft assumes crash faults only, so contrasting the two reveals what Byzantine tolerance costs

## Beliefs

- `pbft-cluster-validates-n-3f-plus-1` — `PBFTCluster` raises `ValueError` if `n` is not exactly `3f+1` or if more than `f` byzantine nodes are specified
- `pbft-duplicate-messages-silently-dropped` — `PBFTNode.receive_message` returns an empty list for duplicate or invalid messages rather than raising exceptions
- `pbft-executed-log-sequence-numbers-contiguous` — Honest nodes execute requests with contiguous sequence numbers starting from 1, enforced by the protocol
- `pbft-view-change-preserves-prepared-requests` — Requests that reach the prepared state survive view changes and are re-proposed by the new primary
- `pbft-submit-request-is-synchronous-simulation` — `PBFTCluster.submit_request` runs the full protocol to completion deterministically (no real networking or async), returning a boolean success indicator

