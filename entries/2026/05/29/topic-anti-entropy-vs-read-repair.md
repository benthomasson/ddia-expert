# Topic: The tradeoffs between lazy read-path repair and background anti-entropy (Merkle tree exchange) for convergence speed and read amplification

**Date:** 2026-05-29
**Time:** 14:05

# Read Repair vs. Anti-Entropy: Convergence Speed and Read Amplification

## The Two Repair Strategies

This codebase implements both repair strategies side by side, which makes the tradeoffs concrete.

### Lazy Read-Path Repair (Read Repair)

Read repair is opportunistic: it piggybacks on reads that are already happening. When a quorum read discovers version divergence among the replicas it contacted, it pushes the newest value to the stale ones before returning.

Look at `ReadRepairStore.get()` in `read-repair/read_repair.py:89-136`. The flow is:

1. Contact R replicas (line 101)
2. Gather responses, find the highest version (lines 104-112)
3. Identify stale replicas **among those R replicas** (lines 122-125)
4. Push the newest value to them (lines 127-130)

The critical detail is on line 100: `targets = available[:self.read_quorum]`. Only the replicas selected for the quorum read are candidates for repair. A replica that wasn't queried — like replica 2 in `test_read_repair_one_stale` (line 38) — stays stale indefinitely until either it's included in a future read or anti-entropy runs.

The `DynamoCluster.get()` in `leaderless-replication/dynamo.py:120-164` does the same thing but goes further: it repairs **all** available nodes, not just the ones in the read quorum (lines 153-161). This is a design choice — more aggressive repair at the cost of higher per-read overhead.

### Background Anti-Entropy (Merkle Tree Exchange)

Anti-entropy is proactive: it runs in the background and synchronizes all replicas regardless of read traffic. The Merkle tree in `merkle-tree/merkle_tree.py` provides the efficient diffing mechanism.

`MerkleTree.diff()` at line 192 compares two trees by walking from the root down. `_diff_recursive()` (line 199) is where the logarithmic efficiency comes from — if a subtree's hash matches, the entire subtree is skipped. Only divergent paths are traversed to the leaf level, yielding the exact set of differing indices.

The actual repair step is in `ReadRepairStore.anti_entropy_repair()` at `read-repair/read_repair.py:141-163` and `DynamoCluster.anti_entropy_repair()` at `leaderless-replication/dynamo.py:180`. Both scan all available replicas, find the max version per key, and push it everywhere stale.

## The Tradeoffs

### Convergence Speed

**Read repair converges only as fast as reads arrive.** A key that's never read stays divergent forever. The test at `read-repair/test_read_repair.py:24-42` demonstrates this: after a write misses replica 2, a read of quorum 2 contacts replicas 0 and 1 — both current — so replica 2 is never repaired. Line 40 confirms: `states[2]["version"] < states[0]["version"]`. Only the explicit `anti_entropy_repair("k1")` call at line 43 fixes it.

**Anti-entropy converges independently of workload.** It can be scheduled at any interval. The tradeoff is that shorter intervals mean faster convergence but more background I/O. The Merkle tree's `O(log N)` diff (via `_diff_recursive` skipping matching subtrees) keeps the comparison cheap — only divergent subtrees are examined — but the actual data transfer for differing keys still scales with the number of inconsistencies.

**Hot keys converge faster under read repair; cold keys may never converge.** Anti-entropy provides a convergence bound that's independent of access pattern.

### Read Amplification

**Read repair increases read-path latency.** Every quorum read must contact R replicas, compare versions, and potentially write back to stale ones. The `DynamoCluster.get()` implementation (dynamo.py:153-161) is particularly expensive — it loops over *all* nodes after every read, not just the quorum participants, turning every read into a potential write fan-out.

The repair stats tracking in `read_repair.py:165-173` quantifies this: `repair_rate` measures what fraction of reads trigger writes. The test at `test_read_repair.py:132-145` shows a 50% repair rate (1 out of 2 reads triggered repair). In a system with frequent node failures, this rate climbs — each read does more work.

**Anti-entropy has zero read-path overhead.** All the comparison and repair work happens in background processes. The cost shifts to background bandwidth and CPU for Merkle tree construction and comparison. The `update_leaf()` method at `merkle_tree.py:129-140` shows the `O(log N)` cost of keeping the tree current as data changes — each write rehashes up the tree to the root.

### The Complementary Design

In practice, both are used together. Read repair handles the common case — hot keys that are frequently accessed converge quickly without background work. Anti-entropy handles the tail — cold keys, keys missed by quorum selection, and replicas that were down for extended periods. The hinted handoff system in `hinted-handoff/hinted_handoff.py` adds a third mechanism specifically for transient failures, with TTL-bounded hints (line 18: `is_expired`) that bridge the gap between failure and recovery.

The `DynamoCluster` shows this layering: `get()` does read repair inline, `anti_entropy_repair()` does full-cluster sweeps, and sloppy quorum with hints (`ReplicaNode.add_hint` at dynamo.py:68) handles the immediate failure window.

## Topics to Explore

- [function] `merkle-tree/merkle_tree.py:_diff_recursive` — The recursive subtree-pruning that makes Merkle-based anti-entropy logarithmic instead of linear
- [function] `leaderless-replication/dynamo.py:get` — Compare this aggressive all-node repair strategy against the quorum-only repair in `read_repair.py:get`
- [file] `hinted-handoff/hinted_handoff.py` — The third repair mechanism (hinted handoff) that bridges transient failures with TTL-bounded write forwarding
- [general] `quorum-intersection-guarantees` — Why R + W > N matters for read repair correctness — the warning at `read_repair.py:53` flags when this invariant is violated
- [function] `merkle-tree/merkle_tree.py:update_leaf` — The O(log N) write-path cost of maintaining a Merkle tree as data changes, which is the ongoing price of enabling efficient anti-entropy

## Beliefs

- `read-repair-only-fixes-queried-replicas` — `ReadRepairStore.get()` only repairs replicas included in the R-sized quorum read; replicas not contacted remain stale until anti-entropy or a future read includes them
- `dynamo-read-repair-is-all-node` — `DynamoCluster.get()` repairs all available nodes after every read, not just quorum participants, trading higher per-read cost for faster convergence
- `merkle-diff-prunes-matching-subtrees` — `MerkleTree._diff_recursive()` returns immediately when subtree hashes match, making the diff proportional to the number of differences rather than total tree size
- `anti-entropy-repair-skips-unavailable` — Both `anti_entropy_repair()` implementations only scan and repair available replicas; a down replica cannot be repaired until it comes back online
- `hinted-handoff-ttl-bounds-hint-lifetime` — Hints expire after `created_at + ttl` (checked via `Hint.is_expired`), so a node that stays down longer than the TTL window will not receive those hinted writes and must rely on anti-entropy for convergence

