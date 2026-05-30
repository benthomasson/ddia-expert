# Topic: Martin Kleppmann's critique of Redis's Redlock algorithm centers on exactly the fencing token mechanism implemented here

**Date:** 2026-05-29
**Time:** 13:04

# Kleppmann's Redlock Critique and the Fencing Token Implementation

## The Core Argument

Kleppmann's central claim against Redlock is simple: **any distributed lock that relies on timing assumptions (TTLs, clocks) can fail, and when it does, you need something else to prevent data corruption.** That "something else" is a fencing token — and Redlock doesn't provide one.

This codebase implements both sides of the argument as a teaching tool.

## How the Code Demonstrates the Problem

### The Lock Service Issues Monotonic Tokens

In `fencing-tokens/fencing_tokens.py:28-40`, `LockService.acquire()` does two things that Redlock does not: it maintains a global counter (`self._counter`, line 25) and attaches a monotonically increasing token to every lock grant. Each acquisition — whether for a new lock or re-acquisition after expiry — increments the counter (line 38). This is the mechanism Kleppmann argues is *necessary* and Redlock *lacks*.

### The Dangerous Scenario: Process Pause After Lock Expiry

The test at `fencing-tokens/test_fencing_tokens.py:108-126` (`test_unsafe_scenario_stale_write_corrupts`) models exactly Kleppmann's attack scenario:

1. Client A acquires a lock (time=0, TTL=5)
2. Client A writes to the resource
3. **Client A pauses** (GC pause, network delay, page fault — any real-world delay)
4. The lock expires at time=5
5. Client B acquires the lock (time=6) and writes
6. Client A wakes up, **still believing it holds the lock**, and overwrites B's data

With `UnfencedResourceServer` (line 100), the stale write succeeds — data corruption. This is the Redlock failure mode. Redlock gives Client A a lock with a TTL, but once A's process pauses past that TTL, there is *nothing* on the storage side to reject the stale write.

### The Fix: Server-Side Token Validation

The companion test at `fencing-tokens/test_fencing_tokens.py:128-145` (`test_safe_scenario_stale_write_rejected`) runs the identical scenario against `FencedResourceServer`. The critical logic is at `fencing_tokens.py:80-85`:

```python
def write(self, resource, key, value, fencing_token):
    highest = self._highest_token.get(resource, 0)
    if fencing_token < highest:
        return {'success': False, 'error': f'Token {fencing_token} is stale ...'}
    self._highest_token[resource] = fencing_token
```

Client B's write arrives with token 2; Client A's belated write carries token 1. The server rejects it. **The safety guarantee moves from the lock (which is time-dependent and fallible) to the storage server (which uses a monotonic comparison and is not time-dependent).**

### Why Renewal Matters

`LockService.renew()` at line 49 deliberately extends the TTL *without incrementing the counter*. This is a subtle design choice: renewal doesn't change the ordering relationship between lock holders. If you issued a new token on renewal, the old holder's token would become artificially "staler" even though no actual ownership change occurred. The test at `test_fencing_tokens.py:68-79` verifies both that the token value is preserved and that the counter stays at 2 (only one acquire happened).

### What This Means for Redlock

Kleppmann's argument is that Redlock provides only the lock — the left side of this implementation. It gives you TTL-based mutual exclusion with no fencing token. When the timing assumption fails (and in distributed systems, it *will* fail — GC pauses, clock skew, network delays), there is no server-side mechanism to catch the stale write. The fix requires **cooperation from the storage layer** (the `FencedResourceServer` pattern), which means the lock alone was never sufficient for correctness — only for efficiency.

The existence of `UnfencedResourceServer` as a separate class makes the pedagogical point explicit: the two implementations are identical except for the token check, and that single `if fencing_token < highest` comparison is the entire difference between data corruption and safety.

## Topics to Explore

- [function] `fencing-tokens/fencing_tokens.py:FencedResourceServer.write` — The `<` comparison (not `<=`) means equal tokens are accepted — explore whether this is intentional and what it implies for idempotent retries
- [general] `clock-dependency-in-lock-expiry` — `is_expired` at line 16 uses injected `current_time` rather than real clocks, sidestepping the very clock-skew problem Kleppmann highlights — worth understanding what breaks if you used `time.time()` instead
- [general] `consensus-vs-fencing` — Kleppmann argues that if you need correctness, you need consensus (e.g., ZooKeeper) not Redlock; explore how ZooKeeper's sequential znodes naturally provide the monotonic fencing token this code manually implements
- [function] `fencing-tokens/fencing_tokens.py:LockService.renew` — Renewal preserves the token value, which means a renewed lock is indistinguishable from the original at the storage server — explore whether this creates an ordering ambiguity if another client acquired and released between renewal checks

## Beliefs

- `fencing-token-counter-is-global` — `LockService._counter` is shared across all lock names, ensuring tokens are globally monotonic — not just monotonic per lock — so a single `FencedResourceServer` can safely validate tokens from different locks
- `fenced-server-rejects-strictly-lower-tokens` — `FencedResourceServer.write` rejects tokens strictly less than the highest seen (`<`, not `<=`), meaning a client can retry with the same token without rejection
- `resource-token-tracking-is-independent` — Each resource in `FencedResourceServer` maintains its own `_highest_token` entry, so a low token on resource B doesn't get rejected because resource A saw a higher one (test 11, line 98)
- `renewal-preserves-token-identity` — `LockService.renew` extends TTL without incrementing the counter or issuing a new token, so the lock holder's ordering position relative to future acquirers is unchanged
- `unfenced-server-exists-as-negative-example` — `UnfencedResourceServer` deliberately omits token validation to demonstrate that without fencing, a stale client can silently corrupt shared state (test 9, line 108)

