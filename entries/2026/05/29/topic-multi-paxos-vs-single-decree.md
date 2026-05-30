# Topic: This implementation runs independent Paxos per slot; compare with a Multi-Paxos leader-lease optimization that skips Phase 1 for a stable leader

**Date:** 2026-05-29
**Time:** 14:23

## Independent Paxos Per Slot vs. Multi-Paxos Leader Lease

### What This Implementation Does

The Total Order Broadcast in `total-order-broadcast/total_order_broadcast.py` runs a **fresh single-decree Paxos instance for every slot** in the log. Each `ConsensusInstance` (line 4) is standalone: it tracks its own `promised` ballot, `accepted_proposal`, and `accepted_value` with no awareness of any other slot.

When a node wants to broadcast a message, `_start_proposal` (line 167) always begins at **Phase 1 (Prepare)**. It creates a new proposal number, initializes a fresh `promises` dict, and sends `prepare` messages to every node — including itself:

```python
# line 167-191
def _start_proposal(self, slot, value, outgoing):
    prop_num = self._make_proposal_number(0)  # always starts at round 0
    self._proposals[slot] = {
        'round': 0,
        'phase': 'prepare',     # <-- always begins here
        ...
    }
    for nid in self.all_ids:
        outgoing.append({'type': 'prepare', ...})
```

This means **every single slot pays the full two-round-trip cost**: Prepare → Promise → Accept → Accepted. For N messages, that's 2N round trips of consensus protocol messages.

### What Multi-Paxos With Leader Lease Would Do Differently

In Multi-Paxos, a stable leader runs Phase 1 **once** and then reuses the resulting ballot number across all subsequent slots. The optimization works like this:

1. **A leader is elected** (via Phase 1 or a separate election mechanism) and receives promises from a majority for some ballot `b`.
2. **For every new slot**, the leader skips directly to Phase 2 (Accept) using ballot `b`. Acceptors already promised not to accept anything below `b`, so the Accept is safe without re-preparing.
3. **A lease timer** prevents other nodes from starting competing Phase 1 rounds while the leader is alive, avoiding unnecessary ballot conflicts.

The cost drops from 2 round trips per slot to **1 round trip per slot** in the steady state — a 50% reduction in latency.

### Where You Can See the Cost in This Code

Look at `_find_proposal_slot` (line 155) and `tick` (line 119). Every pending message triggers `_start_proposal`, which unconditionally enters the `'prepare'` phase. There is no concept of "I already hold promises from a majority, so I can skip ahead." The `_proposals` dict (line 82) tracks per-slot state, but nothing carries over between slots.

The proposal number construction at line 107 also reveals the per-slot independence:

```python
def _make_proposal_number(self, round_num):
    return round_num * self.num_nodes + self.node_id
```

Each slot starts at `round_num=0`. A Multi-Paxos leader would instead use a single monotonically increasing ballot that persists across slots.

### Contrast With the Raft Implementation in This Repo

The Raft implementation in `raft-consensus/raft.py` is essentially Multi-Paxos with leader lease under a different name. Once a node becomes leader (`_become_leader`, line 87), it **never re-runs an election for each log entry**. It simply appends entries to its log (`client_request`, line 148) and replicates them via `AppendEntries` — which is Phase 2 only. The election timeout / heartbeat mechanism (lines 42-46, 161-163) acts as the lease: followers won't start a new election as long as they receive heartbeats.

| Aspect | This TOB (per-slot Paxos) | Multi-Paxos / Raft |
|---|---|---|
| Phase 1 frequency | Every slot | Once per leader term |
| Steady-state round trips | 2 per slot | 1 per slot |
| Leader concept | None — any node proposes | Explicit stable leader |
| Conflict handling | Competing proposals bump rounds | Leader monopolizes appends |
| Implementation complexity | Lower | Higher (lease management, leader failover) |

### Why Per-Slot Paxos Still Matters

The per-slot approach is simpler and correct by construction — each slot is an isolated consensus problem. It's a clearer teaching tool because it shows the full Paxos protocol for every decision. Multi-Paxos is a performance optimization that introduces coupling between slots (the leader's ballot spans all of them), which complicates reasoning about leader failure, ballot recovery, and exactly when a new leader must re-run Phase 1 for in-flight slots.

## Topics to Explore

- [function] `total-order-broadcast/total_order_broadcast.py:_start_proposal` — Trace the full Prepare→Accept lifecycle to understand why skipping Phase 1 requires exactly the leader-lease invariant
- [function] `raft-consensus/raft.py:_become_leader` — See how Raft initializes `next_index`/`match_index` once per term rather than per entry, embodying the Multi-Paxos skip-Phase-1 optimization
- [function] `total-order-broadcast/total_order_broadcast.py:_make_proposal_number` — Understand how proposal numbers encode node identity and round, and why a shared ballot across slots would change this scheme
- [general] `paxos-liveness-dueling-proposers` — This per-slot design is vulnerable to livelock when two nodes propose for the same slot simultaneously; Multi-Paxos leader election solves this
- [function] `raft-consensus/raft.py:_advance_commit_index` — Compare how Raft commits entries (leader counts `match_index` across peers) vs. how TOB counts accept responses per slot

## Beliefs

- `tob-full-paxos-per-slot` — Every slot runs the complete two-phase Paxos protocol from scratch; no state carries between slots to skip Phase 1
- `tob-no-leader-concept` — TOBNode has no leader election or lease mechanism; any node can propose for any undecided slot at any time
- `tob-proposal-number-encodes-node-id` — Proposal numbers are computed as `round * num_nodes + node_id`, guaranteeing uniqueness across nodes within the same round
- `raft-is-multi-paxos-optimization` — The repo's Raft implementation in `raft-consensus/raft.py` embodies the Multi-Paxos leader-lease pattern: one election per term, then Phase-2-only replication for all entries within that term
- `tob-competing-proposals-bump-rounds` — When a slot is decided by another proposer with a different value, the losing node re-proposes its value for a new slot rather than retrying the same slot with a higher round

