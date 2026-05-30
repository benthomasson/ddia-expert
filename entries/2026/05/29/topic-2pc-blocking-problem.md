# Topic: This implementation demonstrates the classic 2PC blocking problem: if the coordinator crashes after `"committing"` is logged but before all participants receive the decision, participants hold locks indefinitely until recovery runs

**Date:** 2026-05-29
**Time:** 14:26

# The 2PC Blocking Problem

## The Core Vulnerability

The blocking problem lives in the gap between two log entries in `Coordinator.execute_transaction()` (`two-phase-commit/two_phase_commit.py:131-138`):

```python
self.log.append({"tx_id": tx_id, "state": "committing"})   # line ~131
for pid in participant_operations:
    p = self.participants[pid]
    if p.is_available():
        p.commit(tx_id)                                      # line ~135
self.log.append({"tx_id": tx_id, "state": "committed"})     # line ~136
```

If the coordinator crashes after logging `"committing"` but before iterating through all participants, some participants never receive the commit decision. Those participants voted `"yes"` during Phase 1 and are stuck in the `"prepared"` state — locks held, unable to proceed.

## Why Participants Can't Help Themselves

Look at `Participant.prepare()` (lines 19-33). When a participant votes yes, two things happen:

1. **Locks are acquired** (line 30): `self.locks[op["key"]] = tx_id`
2. **Operations are buffered** (line 31): `self._pending[tx_id] = operations`

The participant has no authority to unilaterally commit or abort after voting yes. It promised the coordinator it would wait. The `recover()` method on `Participant` (lines 86-91) confirms this — it can only *identify* in-doubt transactions, not resolve them:

```python
def recover(self) -> list[str]:
    """Return list of in-doubt tx_ids (prepared but no commit/abort)."""
    tx_states = {}
    for entry in self.log:
        tx_states[entry["tx_id"]] = entry["state"]
    return [tx_id for tx_id, state in tx_states.items() if state == "prepared"]
```

It returns the list. That's it. The locks on those keys remain held in `self.locks`, blocking any future transaction that touches the same keys (line 26-28 in `prepare()`).

## The Recovery Path

Resolution requires `Coordinator.recover()` (lines 148-173). It scans its own log, finds transactions stuck in `"committing"` or `"aborting"`, and re-sends the decision to any participant still in the `"prepared"` state:

```python
if state == "committing":
    for pid in pids:
        p = self.participants[pid]
        if p.is_available() and p.get_transaction_state(tx_id) == "prepared":
            p.commit(tx_id)
            decisions_resent += 1
```

This is the **only** way to unblock those participants. The test `test_coordinator_recovery` (test file lines 97-110) exercises exactly this scenario: it manually prepares participants, appends a `"committing"` log entry to simulate a crash mid-commit, then calls `recover()` to verify the decision gets delivered.

## The Compound Failure

There's a nastier edge buried here. `Coordinator.recover()` checks `p.is_available()` before re-sending (line 160). If a participant is *also* down when the coordinator recovers, that participant stays locked. Recovery is not retried — it runs once. The participant's locks persist until someone calls `recover()` again after the participant comes back online. Nothing in this implementation does that automatically.

## Topics to Explore

- [function] `two-phase-commit/two_phase_commit.py:Participant.prepare` — Trace how lock acquisition creates the "point of no return" where the participant loses autonomy
- [function] `two-phase-commit/two_phase_commit.py:Coordinator.recover` — Study why recovery only handles `"committing"` and `"aborting"` states but not `"preparing"` — and what happens to transactions stuck in prepare
- [general] `three-phase-commit` — How 3PC adds a pre-commit phase to eliminate the blocking window, and why it still fails under network partitions
- [general] `consensus-vs-2pc` — How Raft/Paxos avoid the single-coordinator SPOF that makes 2PC blocking, and why distributed databases increasingly prefer consensus-based commit protocols
- [file] `two-phase-commit/test_2pc.py` — Notice that no test covers the compound failure (coordinator recovers but a participant is still down), which is the most dangerous real-world scenario

## Beliefs

- `2pc-blocking-window` — The coordinator's commit decision is durable (logged as `"committing"`) before any participant receives it, creating a window where a crash leaves participants locked indefinitely
- `participant-cannot-self-resolve` — `Participant.recover()` identifies in-doubt transactions but cannot resolve them; only the coordinator holds the commit/abort decision
- `recovery-requires-participant-availability` — `Coordinator.recover()` skips unavailable participants, meaning a double failure (coordinator crash + participant crash) leaves locks held until both are up and recovery re-runs
- `locks-block-future-transactions` — A participant in the `"prepared"` state holds key-level locks that cause any subsequent transaction touching the same keys to abort with a lock conflict

