# File: leaderless-replication/dynamo.py

**Date:** 2026-05-29
**Time:** 12:55

## Purpose

`dynamo.py` implements a Dynamo-style leaderless replication system — the core idea from Amazon's 2007 Dynamo paper and Chapter 5 of DDIA. It models a cluster of replica nodes where **any node can accept reads and writes** (no leader), and consistency is maintained through quorum protocols, read repair, anti-entropy, and hinted handoff. This is a teaching implementation: the "network" is simulated in-process via direct method calls, and node failure is modeled by flipping an availability flag.

## Key Components

### `VersionedValue` (dataclass)
A value tagged with a monotonic version number and the node that stored it. This is the unit of storage inside each replica.

### `ReadResult` (dataclass)
Returned by `DynamoCluster.get()`. Carries the resolved value, its version, whether a conflict was detected (multiple distinct values at the same version), and how many replicas were repaired during the read.

### `ReplicaNode`
A single node in the cluster. Owns:
- `_store`: a `dict[str, VersionedValue]` — the local key-value state.
- `_available`: simulates node up/down without actually destroying state (so recovery is realistic).
- `_hints`: a queue of `(key, value, version, target_node_id)` tuples for hinted handoff.

Key contract on `write()`: a write is **accepted only if the incoming version >= the current version** for that key. This means last-writer-wins by version, not by timestamp.

### `DynamoCluster`
The coordinator. Holds the `N`, `W`, `R` quorum parameters, a map of nodes, and a **global per-key version counter** (`_version_counters`). This counter is the single source of version assignment — it lives on the coordinator, not on individual replicas.

## Patterns

**Quorum reads/writes (W + R > N).** The classic Dynamo invariant. The cluster is parameterized by `(n, w, r)` and the caller controls the tradeoff between availability and consistency.

**Read repair.** Every `get()` call opportunistically pushes the latest value to any replica that's behind. This is piggy-backed on the read path — no separate background process needed.

**Anti-entropy repair.** `anti_entropy_repair()` is a full-cluster background sweep: for each key, find the highest version across all available nodes and push it everywhere. This handles keys that haven't been read recently and thus haven't been repaired by read repair.

**Hinted handoff.** When `sloppy_quorum=True` and a write succeeds but some nodes are down, the first available node stores a hint. `deliver_hints()` replays these when the target comes back.

**Simulated failure.** Node availability is a boolean flag. The node's store is preserved when "down," so bringing it back up models a crash-recovery scenario (stale data, not empty data).

## Dependencies

Minimal — only `dataclasses` and `typing` from the stdlib. No external libraries, no inter-module dependencies within the repo. Imported by the two test files (`test_dynamo.py`, `test_dynamo_tester.py`).

## Flow

### Write path (`put`)
1. Increment the global version counter for the key.
2. Fan out to **all** nodes — every available node gets the write.
3. Count acknowledgments. If `ack_count < W`, roll back the version counter and raise `QuorumNotMet`.
4. If sloppy quorum is enabled and some nodes are down, store hints on `available_nodes[0]`.
5. Return the assigned version.

### Read path (`get`)
1. Fan out reads to **all** nodes, collecting `VersionedValue` responses from available ones.
2. Check if the number of available nodes meets `R`. If not, raise `QuorumNotMet`.
3. Find the max version. Check for conflicts: if multiple distinct values exist at the max version, flag `is_conflict=True` and return all distinct values as a list.
4. **Read repair**: push the winning value to any replica with a stale or missing entry.
5. Return `ReadResult`.

### Repair paths
- `anti_entropy_repair()`: iterates all keys across all available nodes, finds the best version, pushes to laggards. Returns total repair count.
- `deliver_hints()`: for each node, pops its hint queue and attempts delivery. If the target is still down, the hint is re-queued on the same node.

## Invariants

1. **Version monotonicity.** `ReplicaNode.write()` only accepts `version >= current.version`. Combined with the global counter in `DynamoCluster`, versions are strictly increasing per key across the cluster.
2. **Quorum gating.** Writes that don't reach `W` acks are rolled back (version counter decremented) and raise. Reads that don't have `R` available nodes raise. This enforces the quorum contract at the coordinator level.
3. **Write fan-out is total.** `put()` sends to every available node, not just `W` of them. The quorum check is on the ack count, not the send count. This maximizes consistency.
4. **Read repair is eager.** Every successful `get()` repairs all stale replicas it can reach — not just enough to meet a quorum.
5. **Hints are single-homed.** All hints for unavailable nodes go to `available_nodes[0]`, regardless of how many nodes are available. This simplifies delivery but creates a hotspot.
6. **Conflict detection is value-based.** Two responses at the same max version with different values trigger `is_conflict`. The conflict is surfaced to the caller as a list, not silently resolved.

## Error Handling

Exactly one exception type: `QuorumNotMet`. Raised by both `put()` and `get()` when the quorum constraint can't be satisfied. On a failed write, the version counter is rolled back to prevent version gaps. No other errors are raised — unavailable nodes silently return `False`/`None` and are skipped.

## Topics to Explore

- [file] `leaderless-replication/test_dynamo.py` — See how quorum parameters, node failures, and conflict scenarios are exercised
- [function] `leaderless-replication/dynamo.py:anti_entropy_repair` — Compare this full-sweep approach with Merkle-tree-based repair in `merkle-tree/merkle_tree.py`
- [file] `hinted-handoff/hinted_handoff.py` — A dedicated implementation of hinted handoff, likely with richer semantics than the inline version here
- [file] `read-repair/read_repair.py` — Compare with the read-repair logic embedded in `DynamoCluster.get()`
- [general] `sloppy-vs-strict-quorum` — The `sloppy_quorum` flag changes write behavior but not read behavior — worth reasoning about the consistency implications

## Beliefs

- `dynamo-write-fans-to-all` — `put()` sends writes to every available node, not just W of them; the quorum check gates on ack count, not send count
- `dynamo-version-counter-is-global` — Version assignment uses a single coordinator-level counter per key (`_version_counters`), not per-replica counters, which prevents version conflicts but requires a centralized coordinator
- `dynamo-read-repair-is-eager` — `get()` repairs all stale replicas reachable during the read, not just enough to satisfy the quorum
- `dynamo-failed-write-rolls-back-version` — When `put()` fails to meet write quorum, it decrements the version counter to prevent version gaps
- `dynamo-hints-single-homed` — Hinted handoffs for all unavailable nodes are stored on `available_nodes[0]` only, creating a single point of responsibility for hint delivery

