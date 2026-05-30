# File: two-phase-commit/test_2pc.py

**Date:** 2026-05-29
**Time:** 14:26

I'll explain the file based on what's visible in the test code and what it reveals about the system under test.

---

## `two-phase-commit/test_2pc.py`

### 1. Purpose

This is the **validation test suite** for a Two-Phase Commit (2PC) implementation — the classic distributed transaction protocol from DDIA Chapter 9. It owns the responsibility of proving that the coordinator and participants correctly implement atomic commit semantics: either all participants commit, or all abort, with no partial state.

The file is structured as 15 standalone test functions that can run both via pytest and as a direct script (`__main__` block). Each test exercises a specific guarantee of the 2PC protocol.

### 2. Key Components

**Single import: `TwoPhaseCommitSystem`** — a facade that wires together a coordinator and a set of named participants. Every test starts by constructing one of these with a list of participant IDs.

The tests break into five logical groups:

| Group | Tests | What they prove |
|-------|-------|-----------------|
| **Happy path** | `test_successful_commit`, `test_committed_durable`, `test_full_lifecycle` | Writes land on the correct participants and persist |
| **Abort semantics** | `test_abort_on_unavailable`, `test_abort_no_state_change`, `test_timeout_unavailable` | Unavailability → abort, and aborted txns leave zero side effects |
| **Locking** | `test_lock_conflict`, `test_locks_released_after_commit`, `test_locks_released_after_abort` | Key-level locking prevents concurrent writes; locks are cleaned up on both commit and abort paths |
| **Recovery** | `test_coordinator_recovery`, `test_participant_recovery` | Crash recovery: coordinator replays its log to finish in-flight commits; participants identify in-doubt transactions |
| **Operations & state** | `test_delete_operation`, `test_concurrent_different_keys`, `test_transaction_log`, `test_get_all_states` | Delete support, non-conflicting key independence, log completeness, state introspection |

### 3. Patterns

- **Facade pattern**: Every test goes through `TwoPhaseCommitSystem`, which hides coordinator/participant wiring. Only `test_coordinator_recovery`, `test_participant_recovery`, and `test_full_lifecycle` reach into internals (`system.coordinator`, `system.participants`).

- **Failure injection via `set_available(False)`**: Rather than simulating network partitions at the transport layer, participants have a boolean availability flag. Tests like `test_abort_on_unavailable` and `test_locks_released_after_abort` toggle this to force the abort path.

- **Manual lock injection** (`test_lock_conflict`): Calls `p.prepare("manual-tx", ...)` directly on a participant to simulate a held lock, then verifies that a full system transaction correctly aborts when it encounters the conflict.

- **Sequential "concurrency"**: `test_concurrent_different_keys` runs two transactions sequentially on different keys. The name is slightly misleading — it tests that the system doesn't carry stale lock state between transactions, not true concurrent execution.

- **Test-as-documentation**: Each test ends with a `print("PASS: ...")` message, and the `__main__` block acts as a runnable specification. This is a pattern common in reference implementations where readability matters more than test infrastructure.

### 4. Dependencies

**Imports:**
- `two_phase_commit.TwoPhaseCommitSystem` — the sole dependency. The system object exposes:
  - `.execute(ops_by_participant)` → `{"outcome": "committed"|"aborted", "reason": ...}`
  - `.coordinator` with `.begin_transaction()`, `.execute_transaction()`, `.recover()`, `.log`, `.get_transaction_state()`
  - `.participants` dict with `.get()`, `.set_available()`, `.prepare()`, `.abort()`, `.recover()`
  - `.get_all_states()` → dict of participant → key/value state

**Imported by:**
- `tester_test_2pc.py` — likely a meta-test or test harness that validates the tests themselves (consistent with the project's `tester_test_*.py` pattern seen in other modules).

### 5. Flow

Every test follows the same three-phase pattern:

1. **Setup**: Create a `TwoPhaseCommitSystem` with named participants, optionally inject failures or pre-existing state.
2. **Execute**: Call `system.execute(...)` with a dict mapping participant IDs to operation lists. Each operation is `{"op": "set"|"delete", "key": ..., "value": ...}`.
3. **Assert**: Check `result["outcome"]`, participant state via `.get()`, and in some cases log contents or recovery results.

The **recovery tests** deviate: `test_coordinator_recovery` manually drives the protocol halfway (begin → prepare → log entries) then calls `recover()` to verify the coordinator replays the commit. `test_participant_recovery` prepares a transaction without committing it, then verifies `recover()` returns the orphaned transaction ID.

### 6. Invariants

The tests collectively enforce these 2PC protocol invariants:

- **Atomicity**: If any participant is unavailable, the entire transaction aborts and no participant's state is modified (`test_abort_no_state_change`).
- **Durability**: Committed values survive and are immediately readable (`test_committed_durable`).
- **Lock exclusivity**: A key locked by one transaction cannot be modified by another (`test_lock_conflict`).
- **Lock cleanup**: Locks are released on both commit (`test_locks_released_after_commit`) and abort (`test_locks_released_after_abort`) paths.
- **Log completeness**: The coordinator log records at least `initiated`, `preparing`, and `committed` states (`test_transaction_log`).
- **Recovery correctness**: A coordinator that crashes after logging `committing` will re-deliver commit decisions to all participants on recovery (`test_coordinator_recovery`).
- **In-doubt identification**: Participants can identify prepared-but-not-committed transactions (`test_participant_recovery`).

### 7. Error Handling

The tests don't test exception paths — the 2PC implementation uses **return-value error signaling** exclusively:

- Failed transactions return `{"outcome": "aborted", "reason": "..."}` rather than raising exceptions.
- `test_timeout_unavailable` asserts the reason string contains either "timeout" or "unavailable", showing the implementation communicates failure cause through the result dict.
- There's no test for invalid operations, malformed input, or coordinator crashes mid-commit (the recovery test simulates a crash by manually constructing log state, not by actually interrupting execution).

---

## Topics to Explore

- [file] `two-phase-commit/two_phase_commit.py` — The implementation behind all these tests: coordinator state machine, participant locking, the log, and the recovery protocol
- [function] `two-phase-commit/two_phase_commit.py:TwoPhaseCommitSystem.execute` — The main entry point: how it orchestrates prepare/commit/abort across participants
- [function] `two-phase-commit/two_phase_commit.py:Coordinator.recover` — How the coordinator replays its write-ahead log to finish in-flight transactions after a crash
- [general] `2pc-blocking-problem` — The classic 2PC limitation where participants holding locks are blocked if the coordinator crashes after prepare but before commit — `test_participant_recovery` hints at this but doesn't fully exercise it
- [file] `two-phase-commit/tester_test_2pc.py` — The meta-test layer: how this project validates that the test suite itself covers the right protocol guarantees

## Beliefs

- `2pc-abort-guarantees-no-side-effects` — When any participant is unavailable, `execute()` returns `"aborted"` and no participant's state is modified, even for participants that were available
- `2pc-uses-key-level-locking` — Participants lock at key granularity during prepare; a second transaction touching a locked key will abort rather than wait
- `2pc-locks-released-on-both-paths` — Locks held by a transaction are released regardless of whether it commits or aborts, preventing deadlocks across sequential transactions
- `2pc-coordinator-recovery-replays-commit` — A coordinator with a `"committing"` log entry will re-send commit decisions to all participants during `recover()`, ensuring crash-after-decision still completes
- `2pc-result-dict-not-exceptions` — The protocol communicates outcomes via `{"outcome": "committed"|"aborted", "reason": ...}` return values, never via exceptions

