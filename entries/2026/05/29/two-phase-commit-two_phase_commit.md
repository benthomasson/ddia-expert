# File: two-phase-commit/two_phase_commit.py

**Date:** 2026-05-29
**Time:** 09:08

# `two-phase-commit/two_phase_commit.py`

## Purpose

This file implements the **Two-Phase Commit (2PC) distributed transaction protocol** as described in DDIA Chapter 9. It provides a self-contained simulation of a coordinator and multiple participant nodes that can atomically commit or abort distributed transactions across participants. The file owns the entire 2PC protocol lifecycle: transaction initiation, prepare/vote collection, commit/abort decision, lock management, and crash recovery.

## Key Components

### `Participant`

A node that holds a local key-value store and participates in distributed transactions.

- **`prepare(tx_id, operations)`** — Phase 1 handler. Checks availability, detects lock conflicts, acquires locks on all keys, stashes operations in `_pending`, and returns a vote (`"yes"` or `"no"`). A single conflicting lock causes a `"no"` vote for the entire transaction.
- **`commit(tx_id)`** — Phase 2 handler for commit. Pops pending operations, applies `set`/`delete` mutations to `self.store`, releases locks. Returns the list of applied operations.
- **`abort(tx_id)`** — Phase 2 handler for abort. Pops pending operations, releases only locks owned by this transaction (defensive check via `self.locks.get(op["key"]) == tx_id`), logs abort.
- **`recover()`** — Scans the log and returns transaction IDs stuck in `"prepared"` state (in-doubt transactions). These need the coordinator to re-send a decision.

### `Coordinator`

Orchestrates the 2PC protocol across participants.

- **`begin_transaction()`** — Allocates a monotonic transaction ID (`tx-001`, `tx-002`, ...) via `itertools.count`.
- **`execute_transaction(tx_id, participant_operations)`** — The core protocol loop. Runs Phase 1 (collect votes), makes the atomic commit/abort decision, then runs Phase 2 (broadcast decision). Returns an outcome dict.
- **`recover()`** — Crash recovery. Scans the coordinator log for transactions stuck in `"committing"` or `"aborting"` states and re-sends the decision to any participant still in `"prepared"` state.

### `TwoPhaseCommitSystem`

Convenience facade that bundles a `Coordinator` and its `Participant` instances. Exposes `execute(operations)` which auto-generates a transaction ID and runs the full protocol. Also provides `get_all_states()` for inspecting all participants' stores.

## Patterns

**In-process simulation** — The coordinator directly holds references to participant objects rather than using network calls. Availability is simulated via `_available` flags, and "timeouts" are modeled as availability checks rather than actual time-based logic.

**Write-ahead logging** — Both coordinator and participants append state transitions to `self.log` before and after performing actions. This enables the `recover()` methods to replay incomplete transactions. The log is append-only; entries are never modified.

**Pessimistic locking** — Participants use exclusive key-level locks (`self.locks` maps key → tx_id). Locks are acquired during `prepare` and released during `commit` or `abort`. There is no deadlock detection — conflicts result in an immediate `"no"` vote.

**All-or-nothing voting** — The coordinator requires unanimous `"yes"` votes. A single `"no"` from any participant causes the entire transaction to abort.

## Dependencies

**Imports:** Only `itertools` (for `itertools.count` — monotonic transaction ID generation). No external dependencies.

**Imported by:** `test_2pc.py` and `tester_test_2pc.py` — the test suite and its meta-test validator.

## Flow

A typical transaction proceeds as:

1. **`TwoPhaseCommitSystem.execute(ops)`** — entry point. Calls `begin_transaction()`, then `execute_transaction()`.
2. **Phase 1 (Prepare):** For each participant in `participant_operations`, the coordinator calls `participant.prepare(tx_id, ops)`. Each participant checks availability, checks for lock conflicts, acquires locks, stashes the operations in `_pending`, and votes `"yes"` or `"no"`.
3. **Decision:** The coordinator checks if `all(v == "yes" for v in votes.values())`.
4. **Phase 2 (Commit or Abort):** The coordinator broadcasts `commit()` or `abort()` to each available participant. Committed participants apply mutations to their store and release locks. Aborted participants discard pending ops and release locks.
5. **Return:** The outcome dict contains `tx_id`, `outcome` (`"committed"` or `"aborted"`), the vote map, and an abort reason if applicable.

## Invariants

- **Unanimous vote required for commit** — `all_yes = all(v == "yes" for v in votes.values())`. One `"no"` vote means abort.
- **Lock exclusivity** — A key can only be locked by one transaction at a time. A second transaction attempting to lock the same key gets a `"no"` vote.
- **Lock ownership on abort** — `abort()` only releases locks where `self.locks.get(op["key"]) == tx_id`, preventing one transaction's abort from releasing another transaction's lock.
- **Log-before-act** — The coordinator logs `"committing"` or `"aborting"` before sending decisions to participants. This is critical for crash recovery: if the coordinator crashes after logging the decision but before all participants receive it, `recover()` can re-send.
- **Transaction ID monotonicity** — IDs are generated by `itertools.count(1)`, guaranteeing unique, ordered IDs within a coordinator instance.

## Error Handling

Errors are **returned as data, never raised as exceptions**. Every method returns a dict with status/reason fields:

- `prepare()` returns `{"vote": "no", "reason": ...}` on lock conflict or unavailability.
- `commit()`/`abort()` return `{"success": False}` when the participant is unavailable.
- `execute_transaction()` returns `{"outcome": "aborted", "reason": ...}` on coordinator unavailability or any `"no"` vote.

Unavailable participants during Phase 2 are silently skipped — the coordinator does not retry or block. This is where `recover()` becomes important: it re-sends decisions to participants that missed them.

There is no timeout implementation despite the `timeout` parameter on `Coordinator.__init__` — it's stored but never used, since availability is modeled as a boolean flag rather than actual network timeouts.

## Topics to Explore

- [file] `two-phase-commit/test_2pc.py` — See how failure scenarios (participant crashes, coordinator crashes, lock conflicts) are exercised against the protocol
- [function] `two-phase-commit/two_phase_commit.py:Coordinator.recover` — Understand crash recovery semantics: which log states trigger re-sends and how in-doubt transactions are resolved
- [general] `2pc-blocking-problem` — This implementation demonstrates the classic 2PC blocking problem: if the coordinator crashes after `"committing"` is logged but before all participants receive the decision, participants hold locks indefinitely until recovery runs
- [file] `raft-consensus/raft.py` — Compare with Raft consensus: 2PC requires a dedicated coordinator while Raft is leaderless-capable, and Raft tolerates minority failures while 2PC requires unanimity
- [general] `three-phase-commit` — Explore 3PC as the non-blocking alternative to 2PC that adds a pre-commit phase to avoid the blocking problem visible in `recover()`

## Beliefs

- `2pc-unanimous-vote` — A transaction commits only if every participant in `participant_operations` votes `"yes"`; a single `"no"` triggers abort for all participants
- `2pc-no-exception-errors` — All error conditions (unavailability, lock conflicts, aborts) are returned as dict values with status fields; no method raises exceptions
- `2pc-timeout-unused` — The `timeout` parameter is accepted by `Coordinator.__init__` but never referenced in any logic; availability is modeled via boolean `_available` flags instead
- `2pc-log-before-decision` — The coordinator appends `"committing"` or `"aborting"` to its log before broadcasting the decision to participants, enabling crash recovery via `recover()`
- `2pc-lock-ownership-guard` — `Participant.abort()` only releases locks where the lock owner matches `tx_id`, preventing cross-transaction lock interference

