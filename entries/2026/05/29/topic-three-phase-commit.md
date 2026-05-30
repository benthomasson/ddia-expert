# Topic: Explore 3PC as the non-blocking alternative to 2PC that adds a pre-commit phase to avoid the blocking problem visible in `recover()`

**Date:** 2026-05-29
**Time:** 14:27

## The Blocking Problem in 2PC and How 3PC Solves It

### The blocking problem lives in `recover()`

The 2PC implementation has two `recover()` methods that reveal the fundamental weakness of the protocol.

**`Participant.recover()`** at `two-phase-commit/two_phase_commit.py:89-93` scans the log for transactions stuck in `"prepared"` state — meaning the participant voted YES but never heard back:

```python
def recover(self) -> list[str]:
    """Return list of in-doubt tx_ids (prepared but no commit/abort)."""
    tx_states = {}
    for entry in self.log:
        tx_states[entry["tx_id"]] = entry["state"]
    return [tx_id for tx_id, state in tx_states.items() if state == "prepared"]
```

A participant in this state is **blocked**. It voted YES, so it promised to commit if asked — it can't unilaterally abort. But it also can't commit without the coordinator's say-so, because other participants might have voted NO. The locks acquired during `prepare()` (line 30-31) remain held, blocking all other transactions that touch those keys. The participant can do nothing except wait.

**`Coordinator.recover()`** at `two-phase-commit/two_phase_commit.py:155-185` handles the coordinator's side — it re-sends commit/abort decisions to participants still in `"prepared"` state. But this only works **if the coordinator comes back**. If it doesn't (permanent crash, network partition), every participant that voted YES is stuck indefinitely.

### Why the participant can't decide on its own

Look at the state machine implied by the coordinator's `execute_transaction()` (lines 109-148):

1. Coordinator logs `"preparing"` and sends PREPARE to all participants
2. Participants vote YES/NO
3. Coordinator logs `"committing"` or `"aborting"` — **this is the single point of decision**
4. Coordinator sends COMMIT or ABORT to all participants

The critical gap: between step 2 (participant votes YES) and step 4 (participant receives decision), the participant holds locks but has no information about the global outcome. If the coordinator dies at step 3, the participant is in-doubt. It can't ask other participants either — another participant might have voted NO and already aborted, or might also be in-doubt.

The test at line 110-119 (`test_participant_recovery`) demonstrates exactly this scenario: a participant prepares but receives no decision, and `recover()` correctly identifies it as in-doubt — but offers no resolution.

### How 3PC eliminates the blocking window

Three-Phase Commit inserts a **pre-commit** phase between the vote and the actual commit, splitting the coordinator's decision into two steps:

| Phase | 2PC | 3PC |
|-------|-----|-----|
| 1 | PREPARE → vote YES/NO | **CanCommit** → vote YES/NO |
| 2 | COMMIT or ABORT | **PreCommit** (all voted YES) or ABORT |
| 3 | — | **DoCommit** |

The key insight: in 3PC, when a participant enters the **pre-commit** state, it knows that **all participants voted YES**. This changes everything about recovery:

- If a participant is in `pre-commit` and the coordinator dies, it can **safely commit** — it knows no participant voted NO.
- If a participant is only in `prepared` (received CanCommit, voted YES, but never got PreCommit), it can **safely abort** — the coordinator hadn't committed to the decision yet.

In the current 2PC code, there's no equivalent safe choice. The `recover()` method at line 89 returns in-doubt transactions but has no mechanism to resolve them without the coordinator. With 3PC, there would be no in-doubt state — every reachable participant state maps to a safe unilateral decision.

### What 3PC would change in this codebase

If you were to extend this implementation to 3PC, the changes would center on:

1. **Splitting `execute_transaction()`**: The coordinator's phase 2 (lines 133-148) would become two phases — first sending PRE-COMMIT to all participants, waiting for acknowledgment, then sending DO-COMMIT.
2. **Adding a pre-commit state to `Participant`**: Between `"prepared"` and `"committed"`, a `"pre-committed"` log entry that records "I know all votes were YES."
3. **Making `Participant.recover()` non-blocking**: Instead of returning a helpless list of in-doubt transactions, it could resolve each one: `"pre-committed"` → commit, `"prepared"` → abort.
4. **Timeout-based resolution**: Adding a timeout at each participant so that if the coordinator is silent for too long, the participant applies its safe default (abort if prepared, commit if pre-committed).

### The tradeoff

3PC eliminates blocking at the cost of an extra round trip (the pre-commit phase) and the assumption of **bounded network delays** — the timeout-based resolution only works if you can distinguish "coordinator is dead" from "message is slow." In an asynchronous network where messages can be delayed arbitrarily, 3PC can still produce inconsistency (one participant times out and aborts while the coordinator eventually sends commit). This is why real distributed systems often prefer 2PC with operational recovery (as in this implementation) or consensus-based commit protocols like Paxos Commit.

## Topics to Explore

- [function] `two-phase-commit/two_phase_commit.py:Coordinator.execute_transaction` — Trace the two-phase state machine to understand exactly where the coordinator's single point of failure sits between the prepare and commit log entries
- [function] `two-phase-commit/two_phase_commit.py:Participant.prepare` — Study how lock acquisition during prepare creates the resource-holding that makes the blocking problem costly, not just theoretically interesting
- [general] `paxos-commit-vs-3pc` — Paxos Commit replaces the coordinator with a consensus group, avoiding both the blocking problem and 3PC's reliance on bounded delays — compare the tradeoffs
- [general] `coordinator-log-durability` — The recovery logic assumes the coordinator's log survives crashes; explore what happens when it doesn't and how WAL/fsync guarantees interact with 2PC correctness
- [file] `two-phase-commit/test_2pc.py` — Write a test that demonstrates the blocking scenario: prepare a participant, kill the coordinator (set unavailable), and show that `recover()` returns in-doubt transactions with no resolution path

## Beliefs

- `2pc-participant-recover-returns-in-doubt-only` — `Participant.recover()` identifies transactions stuck in "prepared" state but provides no mechanism to resolve them without the coordinator, making it a diagnostic tool rather than a recovery procedure
- `2pc-blocking-window-is-between-prepare-and-decision` — A participant that has voted YES in `prepare()` holds locks and cannot unilaterally commit or abort until it receives the coordinator's decision, creating an unbounded blocking window if the coordinator fails
- `2pc-coordinator-recover-requires-liveness` — `Coordinator.recover()` can only re-send decisions to participants that are currently available (`is_available()` check at line 170), leaving unavailable participants still in-doubt
- `2pc-locks-held-during-in-doubt` — Locks acquired during `prepare()` at line 30-31 are only released by `commit()` or `abort()`, meaning an in-doubt transaction blocks all subsequent transactions on the same keys indefinitely

