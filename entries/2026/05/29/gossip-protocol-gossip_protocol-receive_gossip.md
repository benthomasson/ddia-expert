# Function: receive_gossip in gossip-protocol/gossip_protocol.py

**Date:** 2026-05-29
**Time:** 13:06

# `receive_gossip` — Gossip Protocol Membership Merge

## Purpose

`receive_gossip` is the **convergence engine** of the gossip protocol. When a node receives a membership list from a peer, this method merges the remote state into the local membership table. Its job is to ensure that all nodes in the cluster eventually agree on who is alive, suspected, or dead — without a centralized coordinator.

This implements the "receive" half of the epidemic protocol's push-pull exchange. Without it, heartbeat increments and failure detections would stay local to the node that observed them.

## Contract

**Preconditions:**
- `membership_list` is a dict of `{node_id: {heartbeat_counter, timestamp_last_updated, status}}` — the format returned by `send_gossip()`
- `current_time` is monotonically non-decreasing across calls (the protocol uses it for timeout-based failure detection downstream in `detect_failures`)
- The caller has deep-copied the data before passing it in (enforced by `send_gossip()` which returns `copy.deepcopy`)

**Postconditions:**
- Every node in the remote list has been considered for merge
- Local heartbeat counters are monotonically non-decreasing (they only go up)
- Dead status is **sticky** — once a node is marked dead locally, a remote "alive" with a higher heartbeat counter will *not* resurrect it
- No new entry is created for nodes the remote reports as dead if they're unknown locally

**Invariant:** The merge is idempotent with respect to heartbeat counters — receiving the same gossip twice produces the same result.

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `membership_list` | `dict[str, dict]` | Remote node's full membership table. Each value has keys `heartbeat_counter` (int), `timestamp_last_updated` (int), `status` (str: `"alive"`, `"suspected"`, `"dead"`) |
| `current_time` | `int` | Logical clock value for this gossip round. Used to stamp `timestamp_last_updated` on any entry that gets updated, which feeds into `detect_failures`' timeout calculations |

**Edge cases:** If `membership_list` is empty, this is a no-op. If it contains the receiving node's own entry, that entry is merged like any other — there's no self-exclusion guard here (unlike `detect_failures` which skips self).

## Return Value

`None`. All effects are via mutation of `self.membership`.

## Algorithm

The method iterates over every entry in the remote membership list and applies one of three cases:

### Case 1: Unknown node (not in local membership)

```
if nid not in self.membership:
```

- If the remote says the node is **dead**, skip it entirely — no point learning about a node that's already gone.
- Otherwise, deep-copy the remote entry, stamp it with `current_time`, and add it to the local table. This is how nodes discover new cluster members transitively through gossip.

### Case 2: Known node, remote has a higher heartbeat counter

```
if remote["heartbeat_counter"] > local["heartbeat_counter"]:
```

The remote has fresher information. Update the local entry:
1. Adopt the higher heartbeat counter
2. Reset `timestamp_last_updated` to `current_time` (restarts the failure detection clock)
3. Propagate status with an asymmetry:
   - Remote says **dead** → local becomes dead (death propagates unconditionally)
   - Remote says **alive** and local is not dead → local becomes alive (this can clear a "suspected" status)
   - Remote says **alive** but local is **dead** → **no change** (death is sticky, even with a higher counter)

This asymmetry is the critical design choice: once a node is declared dead, it cannot be resurrected by stale "alive" gossip that happens to have a higher counter. This prevents flapping.

### Case 3: Known node, equal heartbeat counter, remote says dead

```
elif (remote["status"] == "dead"
      and remote["heartbeat_counter"] == local["heartbeat_counter"]
      and local["status"] != "dead"):
```

This handles **voluntary leave** (`leave()` sets status to dead without incrementing the heartbeat counter). Without this branch, a node that calls `leave()` would never have its death accepted by peers — they'd see the same counter and ignore the update. This clause accepts the death notification even at equal counters.

### Implicit Case 4: Everything else

If the remote has a lower heartbeat counter, or equal counter without a death notification, the remote info is stale — silently ignored.

## Side Effects

- **Mutates `self.membership` in place** — both existing entries (updating counters, timestamps, status) and adding new entries for previously unknown nodes
- Deep-copies remote entries before insertion to prevent shared mutable state between nodes
- No I/O, no logging, no exceptions raised

## Error Handling

None. The method assumes well-formed input matching the membership entry schema. Missing keys (`heartbeat_counter`, `status`, `timestamp_last_updated`) will raise `KeyError` at runtime. There's no validation — this is internal protocol machinery, not a public API boundary.

## Usage Patterns

Called in two contexts within `GossipCluster.gossip_round()`:

1. **Bidirectional peer exchange** — during normal gossip rounds, two nodes swap membership lists and each calls `receive_gossip` with the other's data
2. **Leave broadcast** — a departing node sends its gossip to all active peers, who merge it via `receive_gossip`

The caller is responsible for providing a consistent `current_time` — the same value should be used for all merges within a single gossip round so that `detect_failures` timeout calculations are coherent.

## Dependencies

- `copy.deepcopy` — used to prevent aliasing between nodes' membership dicts. Without this, two nodes would share the same dict objects and mutations would silently propagate, breaking the simulation's isolation model.
- Relies on the membership entry schema being consistent: `{heartbeat_counter: int, timestamp_last_updated: int, status: str}`. This schema is established in `__init__` and `join` but not enforced by a class or type.

---

## Topics to Explore

- [function] `gossip-protocol/gossip_protocol.py:detect_failures` — The downstream consumer of `timestamp_last_updated`; understanding how it uses the timestamps this method sets clarifies why `current_time` matters
- [function] `gossip-protocol/gossip_protocol.py:leave` — The voluntary leave mechanism that motivates the equal-heartbeat-counter death acceptance clause (Case 3)
- [file] `gossip-protocol/test_gossip_protocol.py` — Test cases that exercise convergence, failure detection, and voluntary leave scenarios
- [function] `gossip-protocol/gossip_protocol.py:gossip_round` — The orchestrator that drives heartbeats, peer selection, and bidirectional exchange — shows how `receive_gossip` fits into the protocol's round structure
- [general] `crdt-convergence-semantics` — The merge function here is a join-semilattice on heartbeat counters with an absorbing "dead" state; understanding CRDTs clarifies why this converges

## Beliefs

- `receive-gossip-death-is-sticky` — Once a node's local status is "dead", `receive_gossip` will never change it back to "alive", even if the remote has a higher heartbeat counter
- `receive-gossip-skips-unknown-dead-nodes` — If the remote reports a node as dead and the local node has never seen that node, no entry is created — preventing the membership table from filling with dead nodes the local node never knew
- `receive-gossip-equal-counter-death-acceptance` — A death notification with an equal heartbeat counter is accepted, specifically to support the voluntary `leave()` protocol where the counter is not incremented
- `receive-gossip-deep-copies-on-insert` — New entries are deep-copied from the remote data before insertion, preventing shared mutable state between nodes in the simulation
- `receive-gossip-no-self-exclusion` — Unlike `detect_failures`, this method does not skip the receiving node's own ID — if the remote gossip contains an entry for the receiver, it is merged normally

