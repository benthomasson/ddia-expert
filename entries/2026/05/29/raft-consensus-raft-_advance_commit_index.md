# Function: _advance_commit_index in raft-consensus/raft.py

**Date:** 2026-05-29
**Time:** 13:50

## `_advance_commit_index` — Raft Leader Commit Advancement

### Purpose

This method implements the leader's commit index advancement rule from the Raft paper (§5.3/§5.4). A leader cannot simply commit an entry the moment it writes it to its own log — it must wait until a majority of the cluster has replicated that entry. This method scans uncommitted log entries, counts how many nodes have replicated each one, and advances `_commit_index` to the highest entry replicated by a majority.

It also enforces a critical safety property: **a leader only commits entries from its own term**, never from a prior leader's term. Entries from prior terms get committed indirectly — once a current-term entry at a higher index is committed, all preceding entries are implicitly committed too. This prevents the scenario in Figure 8 of the Raft paper, where a leader could incorrectly commit an old-term entry that later gets overwritten.

### Contract

**Preconditions:**
- The caller is the leader (`self._state == "leader"`). Both call sites (`tick` and `handle_append_response`) verify this before invoking.
- `self._match_index` reflects the latest known replication state of each peer.
- `self._log` is consistent and indexed starting from a sentinel at position 0.

**Postconditions:**
- `self._commit_index` is monotonically non-decreasing — it either stays the same or advances forward.
- After return, `self._commit_index` is the highest log index `n` where: (a) `self._log[n].term == self._current_term`, and (b) a strict majority of nodes (including the leader) have replicated index `n`.

**Invariant:**
- The commit index never advances past an entry from a prior term, even if that entry is on a majority of nodes.

### Parameters

None — this is a private method that reads directly from instance state:
- `self._commit_index`: current commit watermark
- `self._log`: the replicated log (list of `LogEntry`, 0-indexed with a sentinel)
- `self._current_term`: the leader's current term
- `self._peer_ids`: list of peer node IDs
- `self._match_index`: dict mapping each peer to the highest log index known to be replicated on that peer

### Return Value

None. This method mutates `self._commit_index` as a side effect.

### Algorithm

1. **Iterate candidate indices**: Loop from `commit_index + 1` up through the last log index. Each `n` is a candidate for the new commit index.

2. **Term filter**: Skip any entry whose term doesn't match `_current_term`. This is the safety guard — entries from old terms are never directly committed.

3. **Count replicas**: Start `count` at 1 (the leader always has the entry). For each peer, check if `_match_index[peer] >= n`. If so, that peer has replicated at least up to index `n`, so increment count.

4. **Majority check**: Compute `total = len(peers) + 1` (full cluster size). If `count > total // 2`, a strict majority has replicated index `n`, so advance `_commit_index = n`.

5. **No early exit**: The loop continues even after finding a committable index, because a higher index might also have a majority. The final value of `_commit_index` after the loop is the highest such index.

### Side Effects

- **Mutates `self._commit_index`** — this is the only state change. Once advanced, followers learn the new commit index via the `leader_commit` field in subsequent `AppendEntries` RPCs.

### Error Handling

None. The method assumes all state is well-formed. There are no bounds checks, no exception handling, and no defensive guards. If `_match_index` contains a peer not in `_peer_ids`, that peer is simply ignored (only `_peer_ids` is iterated). If `_match_index` is missing a peer, `dict.get(peer, 0)` defaults to 0, treating that peer as having replicated nothing.

### Usage Patterns

Called in two places:

1. **`tick()`** (line ~139) — on every heartbeat interval, after sending `AppendEntries` to all peers. This catches up the commit index periodically.

2. **`handle_append_response()`** (line ~189) — immediately after a successful replication response updates `_match_index`. This allows the commit index to advance as soon as a majority is confirmed, without waiting for the next heartbeat.

Both callers gate on `self._state == "leader"`, so this method is never invoked on followers or candidates.

### Dependencies

- `self._log` — indexed by integer position; relies on the sentinel at index 0 and contiguous indexing.
- `self._match_index` — populated by `handle_append_response` on successful replication and initialized to all-zeros in `_become_leader`.
- `self._peer_ids` — set at construction, assumed immutable (no membership changes).

### Assumptions Not Enforced by Types

- The log is never empty (sentinel at index 0 is always present).
- `_match_index` values are always valid log indices (never negative, never beyond the log length).
- This method is only called by the leader — no guard exists within the method itself.
- Log indices are contiguous integers matching list positions. If the log were sparse or re-indexed, the `self._log[n]` access would break.
- The loop assumes `_commit_index` is always less than or equal to `_last_log_index()`. If `_commit_index` somehow exceeded the log length, `range()` would produce an empty sequence (safe but silent).

## Topics to Explore

- [function] `raft-consensus/raft.py:handle_append_response` — The other call site for `_advance_commit_index`; shows how `_match_index` is updated and how log backtracking works on failure
- [function] `raft-consensus/raft.py:handle_append_entries` — The follower side: how `leader_commit` propagates the commit index to followers via `min(leader_commit, last_log_index)`
- [general] `raft-figure-8-safety` — The Raft paper's Figure 8 scenario explaining why leaders must not commit entries from prior terms; this is the exact bug the term-check guard prevents
- [file] `raft-consensus/test_raft.py` — Test cases that exercise commit advancement, including partition/heal scenarios that test the majority-counting logic
- [function] `raft-consensus/raft.py:_become_leader` — How `_match_index` and `_next_index` are initialized when a node wins an election, which directly affects the first call to `_advance_commit_index`

## Beliefs

- `raft-leader-only-commits-current-term` — `_advance_commit_index` skips log entries whose term differs from `_current_term`, ensuring old-term entries are only committed indirectly when a current-term entry at a higher index reaches majority
- `raft-commit-index-monotonic` — `_commit_index` is only ever assigned values greater than its current value within `_advance_commit_index`, because the loop starts at `_commit_index + 1` and only assigns forward
- `raft-commit-requires-strict-majority` — An entry is committed when `count > total // 2`, which for a 5-node cluster means 3+ replicas (including the leader), matching Raft's quorum requirement
- `raft-advance-commit-called-twice` — `_advance_commit_index` is invoked both on heartbeat ticks and on successful append responses, so the commit index can advance eagerly without waiting for the next heartbeat cycle
- `raft-match-index-defaults-zero` — Peers missing from `_match_index` are treated as having replicated nothing (`dict.get(peer, 0)`), which is safe because `_become_leader` initializes all peers to 0

