# Topic: DDIA Section 5.4.2 discusses how sloppy quorums provide durability but not the consistency guarantees of strict quorums — a write that "succeeds" via hints may not be readable until handoff completes

**Date:** 2026-05-29
**Time:** 13:13

# Sloppy Quorums: Durability Without Consistency

## The Core Problem

A strict quorum guarantees that any read overlaps with at least one node that saw the latest write (because `W + R > N`). A sloppy quorum breaks this guarantee by counting *hint storage on non-preferred nodes* toward the write quorum — nodes that reads will never consult.

## How the Code Demonstrates This

### The Write Path: Counting Hints as Acks

In `hinted-handoff/hinted_handoff.py:139-141`, the coordinator calculates whether a write "succeeded":

```python
total_acks = len(replicas_written) + len(hints_stored)
success = total_acks >= self.write_quorum
```

This is the critical line. A write with `write_quorum=2` can succeed with 1 real replica write and 1 hint stored on a substitute node. The write is *durable* (it exists on disk somewhere), but not on the nodes that readers will query.

### The Read Path: Only Checks Preferred Nodes

In `hinted-handoff/hinted_handoff.py:148-161`, reads only consult preferred replicas:

```python
def get(self, key: str) -> dict:
    preferred = self.get_preferred_nodes(key)
    results = []
    for nid in preferred:
        node = self.nodes[nid]
        if node.is_available():
            result = node.get(key)
            ...
```

The read never looks at non-preferred nodes. If a write was acknowledged partly through hints on substitute nodes, the read won't find the latest value until `trigger_handoff` (`hinted_handoff.py:170`) delivers those hints back to the recovered preferred replica.

### The Visibility Gap in Action

Consider this scenario with `write_quorum=2`, `replication_factor=3`, preferred nodes `[A, B, C]`:

1. Nodes A and B go down.
2. A write succeeds: C gets the real write, hints are stored on D and E (non-preferred) for A and B. `total_acks = 1 + 2 = 3 >= 2`. Success.
3. A read comes in. It queries preferred nodes `[A, B, C]`. A and B are still down, so only C responds. If `read_quorum=2`, the read *fails entirely*. If `read_quorum=1`, C returns the value — but only because C happened to be available. If C were the one that was down instead, the write would have succeeded via hints but the read would return nothing.

Test `test_sloppy_quorum_succeeds` (`test_hinted_handoff.py:108-117`) exercises exactly this — two preferred nodes down, write succeeds via hints, but no test follows up with a read to show the value is invisible. That's the gap DDIA warns about.

### Contrast: The DynamoCluster Implementation

The `leaderless-replication/dynamo.py` implementation takes a different approach. At line 108, hints are only stored *after* the write quorum is already met on real nodes:

```python
if self.sloppy_quorum:
    unavailable_ids = [nid for nid, n in self.nodes.items() if not n.is_available]
    available_nodes = [n for n in self.nodes.values() if n.is_available]
    for target_id in unavailable_ids:
        if available_nodes:
            available_nodes[0].add_hint(key, value, version, target_id)
```

Here hints are a *bonus* for durability — the quorum was already met by real writes. This is closer to a strict quorum with opportunistic hints, not a true sloppy quorum. The `HintedHandoffStore` implementation is the one that genuinely counts hints toward the quorum threshold.

### Handoff: Closing the Gap

`trigger_handoff` at `hinted_handoff.py:170-193` is what eventually restores consistency. It iterates all nodes, finds hints targeting the recovered node, and replays them:

```python
recovered_node.put(hint.key, hint.value, hint.version)
```

Until this runs, the "successfully written" data is stranded on non-preferred nodes that no reader will ever consult. The test at `test_hinted_handoff.py:55-67` (`test_handoff_delivers_hints`) confirms that the recovered node only has the data *after* handoff, not before.

### Hint Expiry: Durability Has Limits Too

Even the durability guarantee is bounded. Hints have a TTL (`hinted_handoff.py:19-21`), and `expire_hints` removes them. If a node stays down longer than `hint_ttl`, the hints expire and the data is lost from those replicas permanently — only the nodes that received real writes still have it. Test `test_hint_expiry` (`test_hinted_handoff.py:96-106`) validates this with `hint_ttl=10`.

## Topics to Explore

- [function] `hinted-handoff/hinted_handoff.py:HintedHandoffStore.put` — Trace exactly which code paths count hints toward the quorum vs. which require real replica writes
- [file] `read-repair/read_repair.py` — Read repair is the complementary mechanism that fixes staleness on *available* replicas during reads; compare how it restores consistency opportunistically vs. handoff doing it on recovery
- [function] `leaderless-replication/dynamo.py:DynamoCluster.anti_entropy_repair` — The third consistency-restoration mechanism (background full sync), and how it differs from both read repair and hinted handoff
- [general] `sloppy-vs-strict-quorum-test-gap` — The test suite for `HintedHandoffStore` doesn't test reading after a sloppy write but *before* handoff — writing this test would concretely demonstrate the visibility window DDIA describes
- [file] `leaderless-replication/test_dynamo_tester.py` — Contains `test_spec_sloppy_quorum_example` and `test_no_hints_without_sloppy_quorum`, showing how the DynamoCluster's more conservative hint strategy behaves differently

## Beliefs

- `sloppy-quorum-counts-hints-as-acks` — `HintedHandoffStore.put` adds `len(replicas_written) + len(hints_stored)` and compares against `write_quorum`, meaning a write can succeed with zero real replica writes if enough hints are stored
- `reads-never-consult-hint-nodes` — `HintedHandoffStore.get` only queries nodes returned by `get_preferred_nodes`, so data stored only as hints on substitute nodes is invisible to readers until handoff completes
- `dynamo-cluster-hints-after-quorum` — `DynamoCluster.put` only stores hints after the write quorum is already met on real nodes (line 107-113), unlike `HintedHandoffStore` which counts hints toward the quorum
- `hint-expiry-bounds-durability` — Hints in `HintedHandoffStore` have a TTL; if `trigger_handoff` doesn't run before `created_at + ttl`, the hint is silently dropped by `expire_hints`, permanently losing that replica's copy

