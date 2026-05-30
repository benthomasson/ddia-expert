# Topic: This implementation has no pre-vote or leader lease mechanism — explore how split-brain scenarios resolve and whether unnecessary elections occur

**Date:** 2026-05-29
**Time:** 13:51

# Split-Brain Resolution Without Pre-Vote or Leader Lease

## The Missing Safety Mechanisms

The grep results confirm what the question implies: there is **no `PreVote`**, **no `leader_lease`**, and **no `check_quorum`** anywhere in `raft-consensus/raft.py`. These are optimizations described in the Raft dissertation (Section 9.6) and widely implemented in production systems like etcd. Their absence has concrete consequences.

## How Split-Brain Scenarios Actually Resolve

### The Core Safety Guarantee: Term + Majority Vote

This implementation relies entirely on the **original Raft paper's mechanisms** — terms and majority quorum — to prevent true split-brain (two leaders accepting writes in the same term). The critical path is in `handle_request_vote` (lines 99–113):

```python
def handle_request_vote(self, candidate_id, candidate_term, last_log_index, last_log_term):
    if candidate_term > self._current_term:
        self._become_follower(candidate_term)

    if candidate_term < self._current_term:
        return {"term": self._current_term, "vote_granted": False}

    can_vote = (self._voted_for is None or self._voted_for == candidate_id)
    log_ok = self._is_log_up_to_date(last_log_term, last_log_index)
```

Each node votes **at most once per term** (`_voted_for` is set on grant and cleared only when a higher term arrives via `_become_follower` at line 77). Since a candidate needs a majority (`_become_leader` is called in `handle_vote_response`, not shown but implied by `_votes_received`), two leaders in the same term is impossible — you can't get two disjoint majorities from one pool of voters.

### Partition Healing: The Stale Leader Steps Down

When a network partition heals, the stale leader encounters messages with a higher term. Two paths force it to step down:

1. **AppendEntries from the new leader** — `handle_append_entries` lines 115–117:
   ```python
   if leader_term > self._current_term:
       self._become_follower(leader_term)
   ```

2. **RequestVote from a candidate in a higher term** — `handle_request_vote` lines 100–101:
   ```python
   if candidate_term > self._current_term:
       self._become_follower(candidate_term)
   ```

Both call `_become_follower(term)` (lines 76–79), which resets `_voted_for` to `None` and resets the election timer. The stale leader **immediately** gives up leadership upon seeing any message with a higher term.

### The Transient Dual-Leader Window

Here's the subtlety: during a partition, there **can be** two nodes that both believe they are leader — the old leader on the minority side, and the newly elected leader on the majority side. This is not a safety violation because:

- The old leader **cannot commit new entries**. Committing requires replication to a majority (see `_advance_commit_index`, referenced at line 178), and the old leader only has its minority partition.
- The old leader **will accept client requests** (`client_request` at line 162 only checks `self._state != "leader"`), but those entries will never reach `commit_index` and will be overwritten when the partition heals.

This is the window that a **leader lease** would close — with a lease, the old leader would stop accepting client requests once it hadn't heard from a quorum within the lease interval. Without it, clients talking to the stale leader get `{"success": True}` but their writes silently vanish after partition healing.

## Whether Unnecessary Elections Occur

**Yes, and in at least two specific scenarios:**

### 1. Disruptive Rejoining Nodes

When a partitioned node rejoins, it has been incrementing its term through repeated failed elections (line 83: `self._current_term += 1` in `_become_candidate`). When it reconnects, its `RequestVote` messages carry an inflated term. Per line 100–101, the current leader sees this higher term and **immediately steps down**, even though it was functioning perfectly. This triggers a completely unnecessary election across the entire cluster.

This is exactly the scenario `PreVote` prevents. With PreVote, a candidate first asks "would you vote for me?" without incrementing its term. Since the other nodes have a functioning leader (they're receiving heartbeats), they'd refuse the pre-vote, and the rejoining node would never disrupt the cluster.

### 2. Transient Network Delays

If a follower misses heartbeats due to a brief network hiccup (not a real partition), it starts an election (lines 175–186 in `tick`):

```python
self._election_timer += elapsed_ms
if self._election_timer >= self._election_timeout:
    self._become_candidate()
```

The randomized timeout range (default 150–300ms, line 22) provides **some** protection — not all followers time out simultaneously. But a single slow heartbeat delivery can cause a follower to start an unnecessary election, bumping the term and forcing the healthy leader to step down.

### 3. Split Votes and Term Inflation

The test `test_split_vote_resolves` (test file line 168) explicitly exercises this: with a tight timeout range `(150, 155)`, multiple nodes become candidates simultaneously, split the vote, and must retry. Each retry increments `_current_term`. The test passes (`max_ticks=2000`), but the cluster burns through several terms before converging. The test uses `random.seed(42)` to make this deterministic, but in production-like conditions, term inflation during split votes is pure waste.

## The Design Trade-off

This implementation faithfully models the **basic Raft protocol** from the original paper. The omissions are pedagogically sound — PreVote, leader leases, and CheckQuorum add complexity that obscures the core algorithm. But they mean:

| Scenario | With basic Raft (this impl) | With PreVote + Lease |
|---|---|---|
| Partitioned node rejoins | Forces unnecessary election | Silently rejoins |
| Stale leader during partition | Accepts writes that vanish | Stops accepting writes |
| Transient network blip | May trigger election | Heartbeat resume, no election |
| Split vote | Term inflation until resolved | Same (PreVote doesn't help here) |

The `_reset_election_timer()` call on receiving a valid AppendEntries (line 126) is the **only** mechanism preventing unnecessary elections during normal operation — and the test `test_heartbeat_prevents_elections` (test file line 86) verifies this works when the network is healthy.

## Topics to Explore

- [function] `raft-consensus/raft.py:_advance_commit_index` — How the leader determines when a log entry is safely committed by a majority; this is the function that makes the stale-leader-during-partition scenario safe
- [function] `raft-consensus/raft.py:_become_candidate` — The term-increment-on-every-attempt behavior is the root cause of disruptive rejoins; compare with how etcd's PreVote wraps this
- [general] `commit-index-vs-last-applied` — The gap between `_commit_index` and `_last_applied` determines when state machine application happens; understanding this distinction clarifies why uncommitted entries from a stale leader are harmless
- [function] `raft-consensus/raft.py:handle_append_entries` — The log conflict resolution logic (lines 128–152) is what actually repairs divergent logs after a split-brain resolves; trace through the truncation and overwrite paths
- [general] `raft-prevote-extension` — Compare this implementation against etcd's PreVote (Raft dissertation §9.6) to understand the exact message flow that would prevent the disruptive-rejoin problem

## Beliefs

- `raft-no-prevote-or-lease` — The Raft implementation has no PreVote, leader lease, or CheckQuorum mechanism; split-brain safety relies entirely on term numbers and single-vote-per-term invariant
- `raft-stale-leader-accepts-writes` — A partitioned leader continues accepting client requests (returning `success: True`) even though those entries can never commit and will be overwritten on partition heal
- `raft-rejoin-forces-election` — A partitioned node that rejoins after repeated election attempts carries an inflated term, forcing the healthy leader to step down unnecessarily via `_become_follower`
- `raft-randomized-timeout-only-split-vote-defense` — The only mechanism preventing simultaneous candidacy is the randomized election timeout range (default 150–300ms); there is no protocol-level tiebreaker

