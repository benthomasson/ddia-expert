# Topic: How GC pauses and network delays create the exact window fencing tokens protect against (DDIA §8.3)

**Date:** 2026-05-29
**Time:** 13:03

# How GC Pauses and Network Delays Create the Window Fencing Tokens Protect Against

## The Dangerous Window

DDIA §8.3 describes a specific failure scenario that this codebase models directly. The key insight: **a lock's TTL can expire while the lock holder is still alive but unresponsive** — paused by GC, delayed by the network, or simply slow. The holder wakes up believing it still holds the lock and writes to shared state, corrupting data that the new holder is already modifying.

Here's how the code makes this concrete.

## The Timeline, Mapped to Code

**Step 1: Client A acquires the lock and gets token 1.**

In `LockService.acquire` (`fencing_tokens.py:30-37`), the service issues a `FencingToken` with a monotonically increasing counter. Client A receives token `1` and the counter advances to `2`.

**Step 2: Client A enters a GC pause (or hits a network delay).**

This is the critical moment the code *cannot model explicitly* — and that's the point. Nothing in the `Client` class (`fencing_tokens.py:118-153`) tracks whether the client is actually responsive. The client holds a `FencingToken` object in `self._held_tokens` (line 130) and believes it's still valid. There is no heartbeat, no liveness check, no way for the client to know it was paused. The `FencingToken.is_expired` check (line 17) requires passing `current_time` — but a GC-paused process doesn't know time has passed.

**Step 3: The lock expires. Client B acquires the same lock and gets token 2.**

`LockService.acquire` (line 32-33) checks `existing.is_expired(current_time)`. Since `current_time >= issued_at + ttl`, the lock is considered expired, and Client B gets a *new, higher* token. The counter is now `3`.

**Step 4: Client A wakes up and writes with its stale token.**

This is where the two paths diverge — and where the tests make the argument explicit.

### The Unsafe Path: `test_unsafe_scenario_stale_write_corrupts` (test_fencing_tokens.py:107-121)

```python
c_a.acquire_lock("lock", current_time=0, ttl=5)
unfenced.write("shared", "value", "written-by-A")
# Lock expires, B acquires
c_b.acquire_lock("lock", current_time=6, ttl=5)
unfenced.write("shared", "value", "written-by-B")
# A wakes up — stale write succeeds!
unfenced.write("shared", "value", "stale-write-by-A")
```

`UnfencedResourceServer.write` (`fencing_tokens.py:104-108`) accepts every write unconditionally. Client A's stale write overwrites Client B's valid data. The resource server has no way to distinguish a current holder from a zombie.

### The Safe Path: `test_safe_scenario_stale_write_rejected` (test_fencing_tokens.py:123-140)

```python
c_a.acquire_lock("lock", current_time=0, ttl=5)
c_a.write_to_resource(fenced, "shared", "value", "written-by-A", "lock")  # token=1
# Lock expires, B acquires
c_b.acquire_lock("lock", current_time=6, ttl=5)
c_b.write_to_resource(fenced, "shared", "value", "written-by-B", "lock")  # token=2
# A tries stale write — rejected!
result = c_a.write_to_resource(fenced, "shared", "value", "stale-write-by-A", "lock")
assert result['success'] is False
```

`FencedResourceServer.write` (`fencing_tokens.py:83-92`) tracks the highest token seen per resource (`self._highest_token`). When Client A writes with token `1` after the server has already seen token `2`, the comparison at line 87 (`fencing_token < highest`) catches it: token `1 < 2`, write rejected.

## Why the Lock Alone Is Insufficient

The crucial design decision is that **the lock service and the resource server are separate systems** with no shared state. The lock service knows the lock expired. The resource server does not. This separation mirrors real distributed systems where the lock manager (e.g., ZooKeeper) and the storage system (e.g., a database) are independent services.

The fencing token bridges this gap by encoding the lock's causal history *into the write request itself*. The resource server doesn't need to query the lock service — it just compares integers. This is why `_highest_token` is tracked per-resource (line 81) and why `test_independent_resource_token_tracking` (test_fencing_tokens.py:95-101) verifies that different resources maintain independent token counters.

## What the Code Doesn't Show

The grep results confirm that the codebase does **not** model GC pauses or network delays explicitly — there are zero matches for `gc_pause`, `sleep`, `delay`, `frozen`, `network_delay`, or `latency`. The test simulates the *effect* (time jumps from `0` to `6`, skipping over the TTL boundary) rather than the *cause*. This is actually a strength of the design: the fencing token mechanism works regardless of *why* the client was delayed. GC pause, network partition, CPU starvation, operator accidentally suspending the process — the protection is the same.

## Topics to Explore

- [function] `fencing-tokens/fencing_tokens.py:LockService.renew` — Renewal extends TTL without issuing a new token (line 50-57), which means a renewed lock can still be fenced by the *original* token — explore whether this creates a subtle window where renewal races with expiry
- [file] `leader-election/leader_election.py` — The Bully algorithm uses heartbeat timeouts (line 119-120) as the leader liveness detector — compare how heartbeat-based failure detection creates the same ambiguity window that fencing tokens address
- [general] `token-monotonicity-across-lock-names` — Tokens are globally monotonic, not per-lock (line 35 increments a single counter) — explore why this is stronger than per-lock counters and what guarantees it provides
- [function] `fencing-tokens/fencing_tokens.py:FencedResourceServer.write` — The server accepts equal tokens (line 87 uses `<` not `<=`) — explore whether this is intentional and what it means for idempotent retries
- [general] `server-side-token-persistence` — The `_highest_token` dict lives in memory — in a real system, this must survive server restarts, which connects to DDIA's discussion of durable storage for fencing state

## Beliefs

- `fencing-token-counter-is-global` — The `LockService._counter` (line 27) is a single monotonically increasing integer shared across all lock names, not per-lock, ensuring tokens from different locks are totally ordered
- `fenced-server-rejects-strictly-lower-tokens` — `FencedResourceServer.write` rejects writes where `fencing_token < highest` (line 87), but accepts writes with equal tokens, making same-token retries succeed
- `lock-expiry-enables-reacquisition-without-release` — `LockService.acquire` (lines 32-33) checks `is_expired(current_time)` before rejecting a competing acquire, so a crashed client's lock is automatically reclaimable after TTL
- `fencing-tokens-do-not-expire-at-resource-server` — The `FencedResourceServer` has no TTL or expiration logic for tokens — once a token number is seen, all lower tokens are permanently rejected for that resource (test 14, line 180-193)
- `client-holds-stale-token-after-lock-expiry` — The `Client._held_tokens` dict (line 130) is never cleared on lock expiry — the client retains and may use a token whose corresponding lock has already been acquired by another client

