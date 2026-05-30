# Function: _handle_view_change in byzantine-fault-tolerance/pbft.py

**Date:** 2026-05-29
**Time:** 11:50

## `_handle_view_change` — PBFT View Change Handler

### Purpose

This method handles incoming `VIEW_CHANGE` messages, which are the mechanism PBFT uses to replace a faulty or unresponsive primary. When enough nodes agree that the current primary is broken, the system transitions to a new view with a new primary — without losing any requests that were already prepared. This is the liveness guarantee of PBFT: even if the primary fails, the system makes progress.

### Contract

**Preconditions:**
- `msg.msg_type == MessageType.VIEW_CHANGE` (enforced by the caller `receive_message`)
- `msg.sender` is in range `[0, total_nodes)` (validated by caller)
- `msg.data["prepared"]` is a list of dicts with keys `sequence`, `request`, `digest`, `view`

**Postconditions:**
- If this node is the new primary and has collected `2f+1` view-change messages, the view advances to `target_view` and a `NEW_VIEW` message plus re-proposed `PRE_PREPARE` messages are returned
- If this node is *not* the new primary and hasn't yet voted, it echoes its own `VIEW_CHANGE`
- No view change ever moves the view backward

**Invariant:** `self.current_view` is monotonically non-decreasing.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `msg` | `Message` | A `VIEW_CHANGE` message from another node. `msg.view` is the *proposed* new view number, not the sender's current view. `msg.data["prepared"]` carries the sender's prepared-but-uncommitted requests so they can survive the transition. |

### Return Value

Returns `list[Message]` — messages to broadcast. Three possible outcomes:

1. **Empty list** — the message was stale, duplicate, or we're waiting for more votes
2. **Single `VIEW_CHANGE`** — this node is not the new primary but joins the vote
3. **`NEW_VIEW` + `PRE_PREPARE` messages** — this node is the new primary and is taking over, re-proposing any in-flight requests

### Algorithm

**Step 1 — Staleness check.** If `msg.view <= self.current_view`, the view change is for a view we've already moved past. Drop it.

**Step 2 — Deduplication.** Store the message in `view_change_msgs[target_view]`, but only if we haven't already recorded a message from this sender for that view. This prevents a Byzantine node from stuffing the ballot box.

**Step 3 — Branch on role.** The new primary for a view is determined by `target_view % total_nodes`. The method diverges:

**Path A — Not the new primary:**
Check whether we've already sent our own `VIEW_CHANGE` for this view. If not, create one (bundling our own prepared-but-uncommitted data) and broadcast it. This is the "protocol contagion" behavior — receiving a view-change message causes non-primary nodes to join the vote, which is how the quorum builds. If we already sent ours, do nothing.

**Path B — We are the new primary:**
Wait until we have at least `2f+1` view-change messages (the BFT quorum). Once the threshold is met:

1. **Advance the view** — set `self.current_view = target_view`, reset the timer
2. **Merge prepared sets** — union all `prepared` data from every view-change message, deduplicating by sequence number (first writer wins)
3. **Broadcast `NEW_VIEW`** — tells all nodes the view change succeeded
4. **Re-propose requests** — for each prepared-but-uncommitted request from the old view, issue a fresh `PRE_PREPARE` in the new view. This ensures no committed-by-some request is lost during the transition. The sequence counter is bumped if needed to avoid collisions.

### Side Effects

- **`self.current_view`** — advanced to `target_view` (new primary path only)
- **`self.timer_ms`** — reset to 0 (new primary path only)
- **`self.view_change_msgs[target_view]`** — accumulates incoming VC messages; also stores this node's own VC if it creates one
- **`self.message_log`** — re-proposed `PRE_PREPARE` entries are written into the log for the new view
- **`self.accepted_preprepare`** — re-proposed digests are registered, preventing conflicting pre-prepares
- **`self.next_sequence`** — may be bumped forward to avoid sequence collisions with re-proposed requests

### Error Handling

None. The method uses silent drops (return `[]`) for all invalid conditions: stale views, duplicates, insufficient quorum. No exceptions are raised. This is typical for Byzantine protocol implementations — you can't trust the input, so you just ignore what doesn't check out.

### Usage Patterns

Called exclusively from `receive_message` when `msg.msg_type == MessageType.VIEW_CHANGE`. The typical flow:

1. A non-primary node's timer fires → `tick()` calls `_initiate_view_change()` → broadcasts a `VIEW_CHANGE`
2. Other nodes receive it → `_handle_view_change` fires → they echo their own `VIEW_CHANGE` (contagion)
3. The designated new primary collects `2f+1` of these → sends `NEW_VIEW` + re-proposed requests
4. Other nodes process `NEW_VIEW` via `_handle_new_view` → adopt the new view

At the cluster level, `trigger_view_change()` forces this by ticking all non-primary nodes past the timeout.

### Dependencies

- **`_collect_prepared_data()`** — gathers this node's prepared-but-uncommitted requests for inclusion in the VC message
- **`_apply_byzantine()`** — wraps all outgoing messages through the Byzantine behavior filter (silent, equivocating, wrong digest, etc.)
- **`_get_log()`** — creates/retrieves the message log slot for a (view, sequence) pair
- **`compute_digest()`** — re-hashes requests when constructing re-proposed `PRE_PREPARE` messages

### Assumptions Not Enforced by Types

1. **`msg.data["prepared"]` structure** — the code assumes each entry has `sequence`, `request`, `digest`, and `view` keys. A malformed message would raise `KeyError`.
2. **First-writer-wins deduplication on sequence** — when merging prepared sets from multiple VC messages, if two nodes prepared different requests at the same sequence number, only the first one seen survives. The PBFT safety proof guarantees at most one request can be prepared at a given (view, seq), but this code doesn't verify that.
3. **Non-primary nodes don't advance their view** — only the new primary sets `self.current_view = target_view` here. Other nodes update their view when they later receive the `NEW_VIEW` message. Between sending their VC and receiving `NEW_VIEW`, they're in a liminal state where they'll reject normal-phase messages for both the old and new view.
4. **`_apply_byzantine` is called on outgoing but not incoming** — the code trusts that Byzantine behavior is only simulated on send, not on receive. A real PBFT would need MAC/signature verification.

---

## Topics to Explore

- [function] `byzantine-fault-tolerance/pbft.py:_handle_new_view` — The receiving side of the view change: how non-primary nodes adopt the new view and why it's so much simpler than this method
- [function] `byzantine-fault-tolerance/pbft.py:_initiate_view_change` — The trigger that starts a view change; shows how the timer-based failure detector works
- [function] `byzantine-fault-tolerance/pbft.py:_check_prepared` — The quorum check that determines when a request is "prepared" — the 2f threshold that this method relies on for correctness
- [file] `byzantine-fault-tolerance/test_pbft.py` — Test cases that exercise view changes under Byzantine faults, showing the liveness and safety boundaries
- [general] `pbft-view-change-correctness` — Whether the first-writer-wins merge strategy for re-proposed requests preserves the PBFT safety invariant when multiple nodes prepared different requests at the same sequence

---

## Beliefs

- `view-change-requires-2f-plus-1` — The new primary only acts on a view change after collecting at least 2f+1 VIEW_CHANGE messages, matching the PBFT quorum requirement
- `view-change-contagion` — Receiving a VIEW_CHANGE from another node causes a non-primary node to broadcast its own VIEW_CHANGE if it hasn't already, propagating the vote
- `view-never-decreases` — `_handle_view_change` silently drops any message with `msg.view <= self.current_view`, enforcing monotonic view progression
- `reprepare-merges-by-sequence` — When the new primary merges prepared sets from VIEW_CHANGE messages, it deduplicates by sequence number using first-writer-wins, without verifying that all nodes prepared the same request at that sequence
- `only-new-primary-advances-view-in-vc` — Non-primary nodes do not update `current_view` in `_handle_view_change`; they wait for the `NEW_VIEW` message to adopt the new view

