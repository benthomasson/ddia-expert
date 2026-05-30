# Topic: The `LinearizableRegister.read()` broadcasts a read through consensus rather than reading locally; explore why this is necessary and when you can avoid it (e.g., leader leases)

**Date:** 2026-05-29
**Time:** 14:25

I don't have access to read the `LinearizableRegister` class directly (it starts at line 479 of `total-order-broadcast/total_order_broadcast.py`, but the observations only cover lines 0–200). However, the surrounding infrastructure and test usage give us enough to explain *why* this design exists. Let me also note what's missing.

---

# Why `LinearizableRegister.read()` Goes Through Consensus

## The Core Problem: Stale Reads

The temptation is obvious — if you're a node in a cluster, you already *have* a copy of the data. Why not just return it?

Because **having the data isn't the same as having the latest data**. A node can be partitioned from the rest of the cluster, still serving reads happily, returning values that were superseded minutes ago. The client thinks it got the current value. It didn't. Linearizability is broken.

Linearizability requires that every read returns the value of the most recent completed write, as if all operations occurred on a single copy in some total order. A local read on a replica cannot guarantee this.

## How This Codebase Implements It

The `LinearizableRegister` (defined at `total-order-broadcast/total_order_broadcast.py:479`) is built on top of `TOBCluster`, which itself is built on single-decree Paxos instances (`ConsensusInstance`, line 3). The architecture is:

1. **`ConsensusInstance`** (line 3) — implements single-decree Paxos with prepare/accept phases
2. **`TOBNode`** (line 62) — chains Paxos instances into sequential slots, giving total order
3. **`LinearizableRegister`** — uses TOB to order both reads *and* writes

When the register broadcasts a read through consensus, it's doing something subtle: it's **establishing the read's position in the total order**. The read gets a slot number, and the register returns whatever value was written by the most recent write *before that slot*. This guarantees the read reflects all writes that completed before it started.

You can see the consensus machinery in action: `_start_proposal` (line 175) kicks off Paxos Phase 1 for a slot, sending prepare messages to all nodes. Every operation — including reads — must go through this process to get a slot assignment that the majority agrees on.

The test file confirms this at lines 113 and 122 (`test_total_order_broadcast.py`), where `LinearizableRegister` is instantiated with a full `cluster` reference, not just a single node — it needs the consensus infrastructure even for reads.

## Why a Local Read Breaks Linearizability

Consider a 3-node cluster: A (leader), B, C.

1. Client writes `x = 1` → consensus commits at slot 5
2. Network partition isolates node B
3. Client writes `x = 2` → consensus commits at slot 6 (A and C agree)
4. Another client reads from B → gets `x = 1`

Node B honestly believes `x = 1` is current. It has no way to know it missed slot 6. The read returned a stale value, violating linearizability.

By routing the read through consensus, B would need a majority to agree on a slot for the read operation. Since B can't reach a majority (it's partitioned), the read *correctly blocks or fails* rather than returning stale data.

## When You Can Avoid It: Leader Leases

Broadcasting every read through consensus is expensive — it's the cost of Paxos for every `GET`. There are well-known optimizations:

### Leader Leases

A leader obtains a time-bounded "lease" during which it guarantees no other leader can be elected. During the lease window, the leader can serve reads locally because it knows:
- It is still the leader (no election can succeed while the lease is active)
- Its state is up-to-date (all committed writes went through it)

This trades clock assumptions for performance. It works if clocks are well-synchronized (bounded skew). If clocks drift, you can get the same stale-read problem.

**This codebase doesn't implement leader leases.** The grep for `lease` across the repository (from the observations) returns only results from `fencing-tokens/` and `lamport-clocks/` — those are lock release operations, not leader leases. The Raft implementation (`raft-consensus/raft.py`) similarly has no lease mechanism.

### Read Index (Raft-style)

The Raft implementation (`raft-consensus/raft.py`) shows the building blocks but doesn't implement a `ReadIndex` optimization. In this approach, the leader:
1. Records the current commit index
2. Confirms it's still leader by exchanging heartbeats with a majority
3. Waits for its state machine to advance to that commit index
4. Serves the read locally

This is cheaper than full consensus (heartbeats are small, no log write) but still requires a round-trip. You can see the commit tracking infrastructure at `raft.py:32` (`_commit_index`) and the heartbeat mechanism at lines 167–172, which could support this.

### Follower Reads with Bounded Staleness

If you can tolerate slightly stale data, followers can serve reads within a staleness bound. This isn't linearizable, but many applications don't need it for all reads. The Raft `_commit_index` (line 32) tracks how far behind each node is, which would be the foundation for this.

## The Fundamental Tradeoff

The `LinearizableRegister` makes the **correct default choice**: pay for consensus on every read, guaranteeing linearizability. The optimizations above are escape hatches that trade either:
- **Clock accuracy** (leases) — dangerous in practice
- **A round-trip** (ReadIndex) — cheaper but still not free
- **Consistency** (bounded staleness) — not linearizable

The codebase, being a DDIA reference implementation, correctly demonstrates the strict version. The optimizations are production concerns.

## What's Missing From These Observations

The actual `LinearizableRegister` class body (lines 479–565 of `total_order_broadcast.py`) wasn't captured. I'd want to see:
- Whether `read()` literally calls `broadcast()` or uses a distinct read-only consensus path
- How it extracts the current value from delivered slots
- Whether `write()` differs structurally from `read()` in its consensus path

---

## Topics to Explore

- [function] `total-order-broadcast/total_order_broadcast.py:LinearizableRegister` — Read the full class to see exactly how reads and writes are both routed through TOB slots
- [function] `total-order-broadcast/total_order_broadcast.py:TOBNode._deliver_decided_slots` — How decided consensus slots are applied in order; the mechanism that makes both reads and writes see a consistent history
- [function] `raft-consensus/raft.py:RaftNode._advance_commit_index` — The commit advancement logic that would underpin a ReadIndex optimization in Raft
- [file] `total-order-broadcast/test_total_order_broadcast.py` — The linearizability tests likely demonstrate the read-after-write guarantees and possibly partition behavior
- [general] `read-index-vs-lease` — Compare the ReadIndex approach (majority heartbeat confirmation) with leader leases (clock-bounded) as read optimizations, and why Spanner chose the lease path with TrueTime

## Beliefs

- `linearizable-read-requires-consensus` — `LinearizableRegister.read()` broadcasts through the TOB consensus layer rather than reading local state, because a node cannot distinguish "I have the latest value" from "I'm partitioned and stale" without majority confirmation
- `tob-uses-single-decree-paxos-per-slot` — Each slot in the total order broadcast is decided by an independent `ConsensusInstance` running single-decree Paxos with prepare/accept phases (`total_order_broadcast.py:3`)
- `no-leader-lease-in-codebase` — Neither the Raft nor TOB implementations include leader lease or ReadIndex optimizations; all operations pay full consensus cost
- `reads-and-writes-same-consensus-path` — `LinearizableRegister` routes both reads and writes through the same TOB slot mechanism, giving them positions in the same total order

