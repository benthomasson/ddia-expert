# Topic: DDIA Chapter 5 covers LWW, custom merge, and CRDT-based approaches — compare this implementation's strategy enum against the book's taxonomy

**Date:** 2026-05-29
**Time:** 13:55

## DDIA Chapter 5 vs. This Implementation's Conflict Resolution Taxonomy

Kleppmann's Chapter 5 lays out three tiers of conflict resolution for multi-leader and leaderless replication:

1. **Last Write Wins (LWW)** — pick the write with the highest timestamp, discard the other. Simple but lossy.
2. **Custom/application-specific merge** — the application provides a merge function that combines conflicting values (e.g., union of edits, concatenation).
3. **CRDTs** — data structures mathematically designed so that merges are always commutative, associative, and idempotent — conflicts resolve automatically with no data loss.

### What the strategy enum covers

The `ConflictStrategy` enum at `multi-leader-replication/multi_leader.py:10` implements exactly two of these three:

```python
class ConflictStrategy(Enum):
    LAST_WRITE_WINS = "lww"
    CUSTOM_MERGE = "custom"
```

**LWW** (`apply_remote_change`, line ~155): Uses `(timestamp, node_id)` tuple comparison as the total order. The node_id tiebreaker is a standard technique — Kleppmann mentions that LWW needs a total order, and lexicographic node ID is a common way to break timestamp ties. The test at `test_multi_leader.py:76` (`test_lww_tiebreak`) explicitly validates this: when both nodes write at ts=1, node `"b"` beats node `"a"`.

**Custom merge** (`apply_remote_change`, line ~167): The caller passes a `merge_fn(key, local_val, remote_val, local_ts, remote_ts)` callback. This maps to Kleppmann's "application-specific conflict resolution" — the system hands conflicting values to user code. The test at `test_multi_leader.py:21` demonstrates this with a counter-merge that sums the values (`5 + 3 = 8`).

### What's missing: CRDTs as a third strategy

CRDTs exist in the repository as a **separate module** (`conflict-free-replicated-data-types/crdts.py`) but are **not wired into the strategy enum**. The CRDT module implements four classic types:

| CRDT | Line | DDIA connection |
|------|------|-----------------|
| `GCounter` | line 7 | Grow-only counter via per-replica maps |
| `PNCounter` | line 44 | Increment/decrement via two G-Counters |
| `LWWRegister` | line 79 | LWW as a CRDT (note: this is the *CRDT-flavored* LWW, distinct from the multi-leader module's procedural LWW) |
| `ORSet` | line 127 | Observed-remove set with unique tags and tombstones |

This separation reflects an important architectural choice: the multi-leader module treats conflict resolution as a **per-operation decision** (strategy passed to `apply_remote_change`), while CRDTs encode the resolution **into the data structure itself** (every `merge()` method is self-contained). Kleppmann draws this same distinction — CRDTs don't need an external conflict resolver because the merge semantics are baked into the type.

### A notable gap: no vector clocks

The search for `vector.*clock|VectorClock|causal|concurrent` returned zero matches. The multi-leader module uses Lamport timestamps (scalar clock at `multi_leader.py:48`), which provide a total order but **cannot detect concurrency**. Kleppmann emphasizes that Lamport timestamps can't distinguish "A happened before B" from "A and B were concurrent" — you need vector clocks or version vectors for that. The implementation sidesteps this by using `local_origin != remote_node` as a heuristic for concurrency (line ~137), which works for the two-writer case but wouldn't generalize to causal dependency tracking.

### Summary mapping

| DDIA taxonomy | Implementation | Location |
|--------------|---------------|----------|
| LWW | `ConflictStrategy.LAST_WRITE_WINS` | `multi_leader.py:11` |
| Custom merge | `ConflictStrategy.CUSTOM_MERGE` | `multi_leader.py:12` |
| CRDTs | Separate module, not in enum | `crdts.py` (entire file) |
| Vector clocks | Not implemented | — |

The enum covers the "quick and dirty" end of Kleppmann's spectrum. The CRDT module covers the mathematically rigorous end. They're architecturally decoupled — which is honest, since in practice you'd pick one approach per data model, not mix them at the per-operation level.

## Topics to Explore

- [function] `multi-leader-replication/multi_leader.py:apply_remote_change` — The 80-line conflict resolution core; trace how the same-origin vs. different-origin check determines whether a write is an "update" or a "conflict"
- [file] `conflict-free-replicated-data-types/crdts.py` — Compare the `ORSet.merge()` tag-and-tombstone algebra against LWW's timestamp comparison to understand why CRDTs never lose data
- [function] `multi-leader-replication/multi_leader.py:_record_seen` — The idempotency mechanism that prevents reapplication of already-processed changes; critical for ring topology where changes circulate
- [general] `vector-clocks-vs-lamport` — Investigate why the absence of vector clocks limits this implementation's ability to distinguish causal ordering from concurrency, per DDIA Section 5.4
- [file] `multi-leader-replication/test_multi_leader.py` — The test suite is a compact catalog of the conflict scenarios Kleppmann describes; `test_lww_tiebreak` and `test_custom_merge` are particularly instructive

## Beliefs

- `strategy-enum-covers-two-of-three` — `ConflictStrategy` implements LWW and custom merge but not CRDTs; the CRDT module exists separately in `crdts.py` with no integration into the multi-leader replication strategy dispatch
- `lww-tiebreak-uses-node-id` — LWW conflict resolution compares `(timestamp, node_id)` tuples, using lexicographic node ID as a deterministic tiebreaker when timestamps are equal
- `custom-merge-emits-new-timestamp` — `CUSTOM_MERGE` resolution creates a new timestamp (`max(local_ts, remote_ts) + 1`) and a canonical origin, making the merged result a fresh write rather than picking a side
- `no-vector-clocks-anywhere` — The entire repository uses scalar Lamport clocks for ordering; no vector clock or version vector implementation exists, so true concurrency detection is absent
- `crdts-are-self-resolving` — Each CRDT type in `crdts.py` (GCounter, PNCounter, LWWRegister, ORSet) encodes conflict resolution in its own `merge()` method, requiring no external strategy enum or callback

