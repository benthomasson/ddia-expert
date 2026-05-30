# Topic: SWIM is the modern evolution of basic gossip failure detection, adding suspicion-based protocol extensions and piggyback dissemination

**Date:** 2026-05-29
**Time:** 13:08

# SWIM and the Gossip Protocol Implementation

## How This Implementation Relates to SWIM

This gossip protocol implementation in `gossip-protocol/gossip_protocol.py` captures the **suspicion mechanism** from SWIM but omits SWIM's most distinctive feature: **piggyback dissemination**. Understanding both what's here and what's missing is the key takeaway.

### The Suspicion Sub-Protocol (Implemented)

The SWIM paper's central insight is that you shouldn't declare a node dead just because it missed a heartbeat — network hiccups and transient load cause false positives. Instead, nodes transition through an intermediate **suspected** state. This implementation faithfully models that three-phase lifecycle:

```
alive → suspected → dead → removed (cleanup)
```

The timeouts that govern these transitions are defined at `gossip_protocol.py:10`:

```python
def __init__(self, node_id: str, t_suspect: int = 5, t_dead: int = 10, t_cleanup: int = 20):
```

The `detect_failures` method (`gossip_protocol.py:75-101`) is where the suspicion logic lives. It walks the membership list and applies threshold-based transitions:

- **Lines 96-98**: If `elapsed > t_suspect` and the node is still `alive`, mark it `suspected`. This is the SWIM suspicion sub-protocol — the node isn't dead yet, just under observation.
- **Lines 92-95**: If `elapsed > t_dead`, escalate to `dead`. At this point the node is considered permanently failed.
- **Lines 89-91**: If `elapsed > t_cleanup` after death, purge the entry entirely to bound membership list growth.

The suspicion window (`t_suspect` to `t_dead`) is the critical design parameter. Too short, and you get false positives under load. Too long, and genuinely failed nodes linger, consuming resources. The test `test_crash_suspect_dead_cleanup` (`test_gossip_protocol.py:19-42`) validates the full lifecycle.

### Suspicion Recovery via Gossip (Implemented)

A suspected node can be **exonerated**. In `receive_gossip` (`gossip_protocol.py:47-73`), when a remote membership list arrives with a higher heartbeat counter for a suspected node, line 67-68 resets it to `alive`:

```python
elif remote["status"] == "alive" and local["status"] != "dead":
    local["status"] = "alive"
```

This is important: suspicion is reversible, but death is not (via gossip alone). Once a node is marked `dead`, only line 65-66 applies — death propagates but never un-propagates. This matches SWIM's semantics where the `suspected` state is a probationary period, not a permanent mark.

### Piggyback Dissemination (Not Implemented)

The grep for `piggyback` and `disseminat` returned **zero results**. This is the biggest gap between this implementation and the full SWIM protocol.

In SWIM, membership updates (joins, deaths, suspicions) are **piggybacked** onto the protocol's existing ping/ack messages rather than sent in dedicated gossip payloads. This means failure detection and information dissemination share the same bandwidth — you don't need separate gossip rounds to propagate state changes.

This implementation instead uses a **full membership exchange** model. At `gossip_protocol.py:152-158`, each gossip round has nodes swap their entire membership lists:

```python
gossip_from_node = node.send_gossip()      # full deep copy
gossip_from_peer = peer.send_gossip()      # full deep copy
node.receive_gossip(gossip_from_peer, current_time)
peer.receive_gossip(gossip_from_node, current_time)
```

`send_gossip()` (line 46-48) returns a `copy.deepcopy(self.membership)` — the **entire** list, not a delta. This works but is O(N) per exchange rather than O(1) amortized with piggyback.

### What Else SWIM Adds That's Missing

SWIM's failure detection uses **probe-based** detection (ping → indirect ping via k members → suspect), not heartbeat counters. This implementation uses **heartbeat counters with timeouts** (`gossip_protocol.py:38-44`), which is the older gossip-style approach. The probe-based approach is more efficient because it localizes detection cost rather than requiring all nodes to continuously broadcast heartbeats.

## Summary

| SWIM Feature | This Implementation | Location |
|---|---|---|
| Suspicion sub-protocol | Yes — `alive → suspected → dead → removed` | `detect_failures()`, lines 75-101 |
| Suspicion recovery | Yes — higher heartbeat counter clears suspicion | `receive_gossip()`, lines 67-68 |
| Configurable timeouts | Yes — `t_suspect`, `t_dead`, `t_cleanup` | `__init__()`, line 10 |
| Piggyback dissemination | **No** — full membership exchange instead | `send_gossip()`, lines 46-48 |
| Probe-based detection | **No** — heartbeat counters with elapsed time | `heartbeat()`, lines 38-44 |
| Voluntary leave | Yes — marks self dead, broadcasts | `leave()`, lines 31-36 |

This is a solid pedagogical implementation of gossip-based failure detection with the suspicion extension. It demonstrates *why* suspicion matters without the complexity of SWIM's probe protocol or piggyback mechanism.

## Topics to Explore

- [function] `gossip-protocol/gossip_protocol.py:receive_gossip` — The merge logic handles status conflicts (alive vs. dead vs. suspected) with subtle precedence rules that determine whether a node can be exonerated
- [function] `gossip-protocol/gossip_protocol.py:gossip_round` — The orchestration layer reveals this uses push-pull (bidirectional) exchange rather than SWIM's ping/ping-req protocol
- [general] `swim-probe-protocol` — How SWIM's ping → indirect-ping → suspect cycle differs from heartbeat-based detection and why it reduces false positive rates
- [general] `piggyback-dissemination` — How SWIM amortizes dissemination cost by encoding membership deltas into existing protocol messages instead of full-list exchange
- [file] `gossip-protocol/test_gossip_protocol.py` — The O(log N) convergence test (line 81) validates the epidemic dissemination property that both basic gossip and SWIM share

## Beliefs

- `gossip-suspicion-is-timeout-based` — Suspicion is triggered by elapsed time exceeding `t_suspect` since the last heartbeat update, not by failed probe responses as in the SWIM paper
- `gossip-death-is-irreversible-via-gossip` — Once a node's status is `dead`, receiving a gossip message with `status: alive` for that node will never revert it to alive (line 67 checks `local["status"] != "dead"`)
- `gossip-uses-full-membership-exchange` — Each gossip round transmits the entire membership list via `deepcopy`, not deltas or piggybacked updates, making per-exchange cost O(N) in cluster size
- `gossip-cleanup-bounds-membership-growth` — Dead nodes are removed from the membership list after `t_cleanup` elapsed time, preventing unbounded growth of the membership table

