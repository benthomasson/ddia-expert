# Function: _check_prepared in byzantine-fault-tolerance/pbft.py

**Date:** 2026-05-29
**Time:** 11:49

## `_check_prepared` — PBFT Prepared Predicate and Commit Transition

### Purpose

`_check_prepared` implements the **prepared predicate** from the PBFT protocol. It determines whether a request has accumulated enough evidence (a PRE-PREPARE + 2f matching PREPAREs) to transition from the prepare phase to the commit phase. When the predicate is satisfied, the node broadcasts a COMMIT message and checks whether the commit threshold is also met.

This is the gatekeeper between PBFT's second and third phases. Without it, nodes would never advance to committing — and without the quorum check, a Byzantine minority could trick honest nodes into committing inconsistent requests.

### Contract

**Preconditions:**
- `view` and `seq` identify a valid slot in the message log.
- `digest` is the expected hash of the client request, previously validated upstream (in `_handle_pre_prepare` or `_handle_prepare`).
- The caller has already logged the relevant PRE-PREPARE or PREPARE message before calling this method.

**Postconditions:**
- If the prepared predicate is satisfied for the first time: `(view, seq)` is added to `self.prepared_requests`, a COMMIT is logged and returned, and `_check_committed` is invoked.
- If already prepared or the quorum isn't met: returns `[]` with no state changes.

**Invariant:** A given `(view, seq)` pair transitions to "prepared" at most once — enforced by the early-return guard on `self.prepared_requests`.

### Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `view` | `int` | The current view number. Identifies which primary proposed this request. |
| `seq` | `int` | The sequence number assigned by the primary. Orders requests within a view. |
| `digest` | `str` | SHA-256 hex digest of the client request. Used to ensure all nodes agree on the same request content, not just the sequence slot. |

**Edge cases:** If `digest` doesn't match any logged PREPARE messages (e.g., a Byzantine node sent garbage), the `prepare_senders` set stays empty and the quorum check fails harmlessly.

### Return Value

Returns `list[Message]` — either:
- **Empty list** `[]`: predicate not satisfied (missing PRE-PREPARE, insufficient PREPAREs, or already prepared).
- **Non-empty list**: contains the node's own COMMIT message, plus any messages from `_check_committed` (which could include REPLY messages if the commit threshold is also crossed in the same call).

The caller (`_handle_pre_prepare` or `_handle_prepare`) extends its own result list with this return value, so the messages propagate up to `receive_message` and out to the cluster's message bus.

### Algorithm

1. **Idempotency guard** — If `(view, seq)` is already in `prepared_requests`, return early. This prevents duplicate COMMIT broadcasts.

2. **PRE-PREPARE check** — Verify at least one PRE-PREPARE exists in the log for this slot. Without the primary's proposal, there's nothing to prepare.

3. **Quorum count** — Iterate all logged PREPARE messages, collect senders whose digest matches. Uses a set to deduplicate — only distinct senders count. This is critical: without dedup, a Byzantine node could spam PREPAREs to inflate the count.

4. **Threshold check** — Require `2f` distinct matching PREPAREs. In a 3f+1 system, this means the primary (who sent PRE-PREPARE, not PREPARE) plus 2f preparers form a quorum of 2f+1 nodes that agree on the request. Note: the primary is excluded from the PREPARE count by convention — `_handle_prepare` rejects PREPAREs from the primary, and the primary's `_handle_pre_prepare` path skips sending a PREPARE.

5. **Mark prepared** — Add `(view, seq)` to `prepared_requests`.

6. **Create and log COMMIT** — Build a COMMIT message from this node, append it to the local log, and register the node in `phase_senders` so future self-COMMITs are rejected as duplicates.

7. **Chain to commit check** — Call `_check_committed` to see if the commit threshold (2f+1 COMMITs) is already met. This handles the case where COMMIT messages from other nodes arrived before this node reached prepared.

### Side Effects

| Mutation | What changes |
|----------|-------------|
| `self.prepared_requests` | `(view, seq)` added — marks the slot as prepared |
| `self.message_log` | Own COMMIT appended to the commit log for this slot |
| `self.phase_senders` | Own node_id added to the COMMIT sender set — prevents self-dedup issues |
| Transitive via `_check_committed` | May add to `committed_requests`, `_executed_log`, and advance `next_execute_seq` |

No I/O or exceptions. All mutations are in-memory.

### Error Handling

None — the method uses early returns for all failure conditions. No exceptions are raised or caught. Invalid states (wrong digest, missing messages) result in a silent `[]` return. This is deliberate: in a Byzantine protocol, you don't tell potentially-faulty peers why you rejected their input.

### Usage Patterns

Called from exactly two places:

1. **`_handle_pre_prepare`** — After a backup logs the PRE-PREPARE and its own PREPARE, or after the primary logs its own PRE-PREPARE (primary skips PREPARE).
2. **`_handle_prepare`** — After logging each incoming PREPARE message. Since each new PREPARE might push the count past 2f, every arrival triggers a re-check.

Both callers pass the digest they've already validated, so `_check_prepared` trusts it. The callers append the returned messages to their own result lists.

### Dependencies

- **`_get_log`** — Retrieves (or initializes) the per-slot message log.
- **`_check_committed`** — The next phase check, chained at the end. Mirrors this method's structure but with a 2f+1 threshold.
- **`Message`** — Constructs the outgoing COMMIT.
- **`self.prepared_requests`** — Set-based idempotency guard shared across all callers.
- **`self.phase_senders`** — Duplicate-sender tracking shared with `_is_duplicate`.

### Subtle Design Choices

The threshold is `2f`, not `2f+1`. This is because the primary's PRE-PREPARE counts as implicit agreement — the primary proposed this request, so it doesn't need to send a redundant PREPARE. The total agreement is PRE-PREPARE(1) + PREPAREs(2f) = 2f+1, which is the standard PBFT quorum. The code in `_handle_prepare` enforces this asymmetry by rejecting PREPAREs from the primary.

The method logs its own COMMIT before broadcasting, and manually registers in `phase_senders`. This is necessary because when the COMMIT eventually arrives back at this node (via the cluster bus), `_is_duplicate` will correctly reject it as already seen.

---

## Topics to Explore

- [function] `byzantine-fault-tolerance/pbft.py:_check_committed` — The mirror of `_check_prepared` for the commit phase; uses 2f+1 threshold instead of 2f, and chains to execution
- [function] `byzantine-fault-tolerance/pbft.py:_handle_pre_prepare` — Shows the two code paths (primary vs backup) that lead into `_check_prepared`, and why the primary skips sending PREPARE
- [function] `byzantine-fault-tolerance/pbft.py:_apply_byzantine` — How Byzantine fault modes (equivocating, wrong digest, silent) corrupt the messages that `_check_prepared` returns
- [file] `byzantine-fault-tolerance/test_pbft.py` — Test cases that exercise the quorum logic, including Byzantine scenarios where `_check_prepared` must not be fooled
- [general] `pbft-prepared-vs-committed-predicates` — The formal distinction between "prepared" (2f+1 agree on ordering) and "committed-local" (2f+1 will commit) in Castro & Liskov's original paper

---

## Beliefs

- `pbft-prepared-threshold-is-2f` — `_check_prepared` requires exactly 2f matching PREPAREs (not 2f+1) because the primary's PRE-PREPARE counts as implicit agreement toward the quorum of 2f+1
- `pbft-prepared-at-most-once` — A `(view, seq)` pair can transition to "prepared" at most once per node, enforced by the `prepared_requests` set guard at the top of `_check_prepared`
- `pbft-prepare-digest-match-required` — Only PREPARE messages whose digest matches the argument are counted toward the quorum; mismatched digests (from Byzantine nodes) are silently excluded
- `pbft-commit-self-logged-before-broadcast` — `_check_prepared` logs the node's own COMMIT into `message_log` and `phase_senders` before returning it for broadcast, so the node's duplicate detection correctly rejects re-delivery of its own COMMIT
- `pbft-prepared-chains-to-committed` — `_check_prepared` always calls `_check_committed` after marking prepared, enabling immediate commit if enough COMMIT messages arrived out of order

