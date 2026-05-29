# Topic: How PBFT uses `compute_digest` to handle integrity in a fully adversarial environment where nodes may lie

**Date:** 2026-05-29
**Time:** 08:12

# How PBFT Uses `compute_digest` to Handle Integrity in an Adversarial Environment

## The Core Problem

In PBFT, any node might be lying. A faulty primary could propose one operation to node A and a different operation to node B (equivocation). A faulty replica could claim it received a request it never saw, or confirm a request it actually rejected. The protocol needs a way for honest nodes to verify they're all talking about the *same* request — without trusting anyone's word for it.

## The Digest as a Content Fingerprint

The function at `byzantine-fault-tolerance/pbft.py:44` is deceptively simple:

```python
def compute_digest(request) -> str:
    return hashlib.sha256(json.dumps(request, sort_keys=True, default=str).encode()).hexdigest()
```

It takes an arbitrary request, serializes it deterministically (`sort_keys=True` eliminates key-ordering ambiguity), and produces a SHA-256 hash. This hash acts as a **binding commitment** to the exact content of a request. Two nodes that independently compute the digest of the same request will always get the same result. Two different requests will (with overwhelming probability) produce different digests.

This is critical because PBFT messages carry the digest, not the full request. When a node says "I'm prepared for digest `abc123` at sequence 5," every other honest node can independently verify whether `abc123` actually corresponds to the request they received.

## Where the Digest Gets Checked

### 1. Pre-Prepare Validation (Lines ~168–174)

When a backup replica receives a PRE_PREPARE from the primary, it doesn't just trust the primary's claimed digest:

```python
request = msg.data.get("request")
expected_digest = compute_digest(request)
if msg.digest != expected_digest:
    return []
```

The node recomputes the digest from the actual request data bundled in the message. If the primary attached the wrong digest — whether by bug or malice — the message is silently dropped. This prevents a Byzantine primary from binding a legitimate-looking digest to a malicious payload.

### 2. Conflicting Pre-Prepare Detection (Lines ~177–181)

```python
key = (msg.view, msg.sequence)
if key in self.accepted_preprepare:
    if self.accepted_preprepare[key] != msg.digest:
        return []
self.accepted_preprepare[key] = msg.digest
```

Once a node accepts a pre-prepare for a given `(view, sequence)`, it records the digest. If it later receives a *different* digest for the same slot, it rejects it. This is the anti-equivocation guard: a Byzantine primary that sends different requests to different nodes with the same sequence number will be caught, because the digests won't match.

### 3. Prepare and Commit Phase Agreement

In `_check_prepared` and `_check_committed` (not fully shown but implied by the structure), nodes count how many PREPARE and COMMIT messages they've received **for a specific digest**. The quorum checks (2f+1 for commit, 2f for prepare) only count messages whose digest matches the one from the accepted pre-prepare. A Byzantine node that sends a PREPARE with a wrong digest (as simulated by `ByzantineMode.WRONG_DIGEST` at line 121) simply doesn't count toward quorum.

## How Byzantine Modes Test This

The code models exactly the attacks that digests defend against:

| Mode | Attack | How digest stops it |
|------|--------|-------------------|
| `WRONG_DIGEST` (line 121) | Sends `"bad_digest_" + node_id` | Honest nodes recompute and reject the mismatch |
| `EQUIVOCATING` (line 123) | Sends different digests to different peers | Each peer independently verifies against the request; conflicting digests can't reach quorum |

The test at `test_pbft.py:51` (`test_wrong_digest_byzantine`) confirms that even with a node spewing bad digests, honest nodes still reach agreement. The test at line 44 (`test_equivocating_byzantine`) confirms that equivocation — sending `compute_digest(f"equivoc_{peer}_{m.sequence}")` to each peer — also fails to disrupt consensus.

## The Deeper Design Insight

The digest transforms a **trust problem** into a **math problem**. Instead of asking "did node 3 tell the truth about the request?", each node asks "does the SHA-256 of the request I have match the digest in this message?" No node's honesty matters for this check — it's a deterministic computation any node can perform independently. The protocol then uses quorum intersection (2f+1 out of 3f+1 nodes) to guarantee that at least f+1 honest nodes agreed on the *same* digest, which means they agreed on the *same* request.

## Topics to Explore

- [function] `byzantine-fault-tolerance/pbft.py:_check_prepared` — How the quorum logic counts matching digests to determine when a request is "prepared" (2f matching PREPAREs)
- [function] `byzantine-fault-tolerance/pbft.py:_handle_view_change` — How digests carry over during view changes to ensure previously-prepared requests aren't lost or tampered with
- [function] `byzantine-fault-tolerance/pbft.py:_apply_byzantine` — The full catalog of simulated Byzantine behaviors and how each one is neutralized by protocol mechanisms
- [general] `json-canonicalization-risks` — Whether `json.dumps(sort_keys=True)` is truly canonical (it isn't for floats, nested dicts with mixed types, or non-ASCII) and when this matters
- [file] `byzantine-fault-tolerance/test_pbft.py` — The test for 7-node clusters with 2 Byzantine faults (line 103) stress-tests the f < n/3 bound

## Beliefs

- `digest-recomputed-on-preprepare` — Backup replicas independently recompute the digest from the request payload in every PRE_PREPARE; they never trust the primary's claimed digest
- `digest-binds-view-sequence-to-request` — The `accepted_preprepare` dict enforces a one-to-one mapping from `(view, sequence)` to digest, preventing a Byzantine primary from assigning two different requests to the same slot
- `wrong-digest-cannot-reach-quorum` — Messages with non-matching digests are silently dropped and never count toward prepare or commit quorum thresholds
- `deterministic-serialization-enables-independent-verification` — `json.dumps(sort_keys=True)` ensures all honest nodes produce identical digests for identical requests without any inter-node coordination

