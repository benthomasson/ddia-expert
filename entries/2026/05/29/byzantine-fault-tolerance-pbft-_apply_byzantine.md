# Function: _apply_byzantine in byzantine-fault-tolerance/pbft.py

**Date:** 2026-05-29
**Time:** 11:50

# `_apply_byzantine` — Byzantine Fault Injection for PBFT Messages

## Purpose

`_apply_byzantine` is a message-corruption filter that sits between a node's protocol logic and the network. Every outgoing message batch passes through it before transmission. For honest nodes it's a no-op passthrough; for Byzantine nodes it mutates, drops, or multiplies messages to simulate specific classes of faulty behavior that PBFT is designed to tolerate.

This exists to make the simulation testable — rather than writing separate "bad node" implementations, the same `PBFTNode` class handles all roles. The Byzantine mode is set at construction time and applied uniformly to all outbound traffic.

## Contract

- **Precondition**: `self.byzantine_mode` is one of the five `ByzantineMode` constants. The method doesn't validate this — an unknown mode silently falls through to the final `return messages`.
- **Postcondition**: The returned list contains zero or more `Message` objects. For `SILENT`, the list is always empty. For `EQUIVOCATING`, the list may be *larger* than the input. For all other modes, the list has the same length as the input.
- **Invariant**: The method never modifies `self` — it only mutates the `Message` objects passed in (which the caller just created) or creates new ones.

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `messages` | `list[Message]` | Outgoing messages produced by the node's protocol handlers. These are freshly constructed by the caller, so mutating them in-place is safe. |

Edge case: an empty list is valid input and returns an empty list for all modes.

## Return Value

A `list[Message]` that the caller (`receive_message`, `submit_request`, `tick`, `_handle_view_change`) forwards to the cluster's message bus. The caller does not inspect or filter the result — it trusts this method completely, which is the point.

## Algorithm

The method dispatches on `self.byzantine_mode` through a chain of early-return `if` blocks:

1. **HONEST** — Return messages unchanged. This is the fast path and is checked first.

2. **SILENT** — Return an empty list. The node produces valid internal state (it still logs messages, advances sequences) but nothing leaves. This simulates a crash fault or a node that simply stops participating.

3. **WRONG_SEQUENCE** — Add 1000 to every message's sequence number. Honest nodes will reject these because the sequence won't match any accepted pre-prepare. This tests that nodes validate sequence numbers rather than blindly trusting them.

4. **WRONG_DIGEST** — Replace every message's digest with a deterministic bad value (`"bad_digest_{node_id}"`). This tests that nodes verify digest consistency across the pre-prepare/prepare/commit chain. The node-specific string ensures different Byzantine nodes produce different bad digests.

5. **EQUIVOCATING** — The most sophisticated mode. For each broadcast message (`recipient is None`), it creates *N-1* targeted copies, one per peer, each with a **different digest** computed from `f"equivoc_{peer}_{sequence}"`. This means every peer receives a message that appears valid but disagrees with what every other peer received. Already-targeted messages pass through unchanged. This is the classic equivocation attack: a Byzantine primary sends conflicting pre-prepares to different replicas, trying to split the quorum.

6. **Fallthrough** — If none of the modes match (shouldn't happen with the defined constants), return messages unchanged.

## Side Effects

- **Mutates input messages in-place** for `WRONG_SEQUENCE` and `WRONG_DIGEST`. This is safe because callers always construct fresh `Message` objects before calling, but it means the method is not pure.
- **Creates new `Message` objects** for `EQUIVOCATING` via `dict(m.data)` (shallow copy of data dict). The original messages in the broadcast case are discarded — they don't appear in the output.
- No I/O, no state changes on `self`.

## Error Handling

None. No exceptions are raised or caught. Invalid modes silently return the original messages. There's no logging or alerting when Byzantine behavior is applied — this is intentional for a simulation.

## Usage Patterns

The method is called in exactly four places, always wrapping the output of a protocol handler:

```python
# In receive_message:
return self._apply_byzantine(self._handle_pre_prepare(message))
return self._apply_byzantine(self._handle_prepare(message))
return self._apply_byzantine(self._handle_commit(message))

# In submit_request:
result = self._apply_byzantine([pp])

# In tick:
return self._apply_byzantine(self._initiate_view_change())

# In _handle_view_change:
return self._apply_byzantine([vc])
```

Note that `_handle_view_change` also calls `_apply_byzantine` when re-proposing prepared requests during a view change — Byzantine behavior is applied consistently to *all* outbound paths.

Caller obligation: the caller must not reuse the message objects after passing them in, since `WRONG_SEQUENCE` and `WRONG_DIGEST` mutate them.

## Dependencies

- `ByzantineMode` — enum-like class providing the mode constants
- `Message` — the message data class, constructed directly in the `EQUIVOCATING` branch
- `compute_digest` — used in `EQUIVOCATING` to generate per-peer digests that are syntactically valid SHA-256 hashes (not just garbage strings), making the equivocation harder to detect by format alone

## Assumptions Not Enforced by Types

1. `self.byzantine_mode` is assumed to be one of the five `ByzantineMode` constants, but it's a plain `str` — any string is accepted at construction.
2. `self.total_nodes` and `self.node_id` are assumed valid (non-negative, node_id < total_nodes) — used in `EQUIVOCATING` to iterate peers.
3. Messages in the input list are assumed to be freshly created and not shared with other references — the in-place mutation of `WRONG_SEQUENCE`/`WRONG_DIGEST` would corrupt shared state otherwise.
4. `dict(m.data)` in the `EQUIVOCATING` branch is a shallow copy — if `data` contains nested mutable objects, the equivocated messages would share references with the original.

---

## Topics to Explore

- [function] `byzantine-fault-tolerance/pbft.py:_check_prepared` — The quorum-counting logic that Byzantine messages are designed to defeat; understanding what "2f matching PREPAREs" means clarifies why each attack mode fails or succeeds
- [function] `byzantine-fault-tolerance/pbft.py:_handle_view_change` — View changes are the recovery mechanism when a Byzantine primary is detected; trace how `_apply_byzantine` is applied to the new-view protocol path
- [file] `byzantine-fault-tolerance/test_pbft.py` — The test suite exercises each Byzantine mode against the cluster, showing which attacks PBFT tolerates and which safety properties hold
- [general] `pbft-equivocation-defense` — How the prepare/commit two-phase structure prevents equivocating primaries from causing honest nodes to disagree, even though each sees a different digest
- [general] `byzantine-fault-budget` — The `3f+1` constraint enforced in `PBFTCluster.__init__` and why each attack mode requires at most `f` faulty nodes to be tolerated

## Beliefs

- `apply-byzantine-is-outbound-only` — `_apply_byzantine` is applied exclusively to outgoing messages; incoming messages are never filtered through it, so a Byzantine node's internal state remains consistent even while it lies to peers
- `equivocating-mode-expands-message-count` — In `EQUIVOCATING` mode, each broadcast message becomes N-1 targeted messages with distinct digests, meaning the output list can be larger than the input
- `silent-mode-drops-all-outbound` — `SILENT` mode returns an empty list regardless of input, but the node still processes incoming messages and updates internal state (prepared/committed sets)
- `wrong-digest-is-node-specific` — `WRONG_DIGEST` produces `"bad_digest_{node_id}"`, so two Byzantine nodes with this mode produce different invalid digests rather than colluding on the same one
- `byzantine-mode-is-immutable-after-construction` — The `byzantine_mode` field is set in `__init__` and never modified, so a node cannot change its fault behavior mid-protocol

