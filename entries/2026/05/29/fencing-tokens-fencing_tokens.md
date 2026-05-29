# File: fencing-tokens/fencing_tokens.py

**Date:** 2026-05-29
**Time:** 08:54

## Purpose

This file implements **fencing tokens for distributed locking**, the mechanism described in DDIA Chapters 8–9 for preventing stale lock holders from corrupting shared resources. The core problem: a client acquires a lock, gets paused (GC, network delay), the lock expires, another client acquires it, and now two clients think they hold the lock. Fencing tokens solve this by attaching a monotonically increasing integer to each lock acquisition — resource servers reject writes from tokens lower than the highest they've seen.

The file provides all four actors in this protocol: the token itself, the lock service that issues them, the resource servers (fenced and unfenced for comparison), and the client that ties them together.

## Key Components

### `FencingToken`
A value object representing an issued lock grant. Carries five fields: the monotonic `token` integer, `lock_name`, `client_id`, `issued_at` timestamp, and `ttl`. The only behavior is `is_expired(current_time)`, which checks `current_time >= issued_at + ttl`. Note the `>=` — expiry is inclusive of the boundary, meaning a token with `issued_at=0, ttl=10` is expired at time 10.

### `LockService`
The central lock authority. Maintains a global `_counter` (starts at 1) and a map of lock names to their current `FencingToken`.

- **`acquire()`** — The grant logic has three paths: (1) lock doesn't exist → grant, (2) lock exists but expired → grant, (3) lock held by same client → grant (re-entrant). Only blocks if a *different* client holds an unexpired lock. Every successful acquisition increments the counter and issues a new token, even re-entrant ones.
- **`release()`** — Only the current holder can release. Deletes the lock entry entirely.
- **`renew()`** — Extends TTL *without* issuing a new token number. This is a deliberate design choice: renewal preserves the original fencing token value, so ongoing writes from the same session remain valid. Fails if the lock is expired or held by someone else.
- **`is_held()`** — Read-only query returning holder info or `None`.

### `FencedResourceServer`
A key-value store organized as `resource → key → value`. The critical invariant: it tracks `_highest_token` per resource and rejects any write whose `fencing_token < highest`. This is the server-side enforcement that makes fencing work — a slow client whose lock expired and was re-acquired by another client will be rejected when it finally tries to write.

### `UnfencedResourceServer`
Identical storage structure but accepts all writes unconditionally. Exists as a **pedagogical counterexample** — tests can demonstrate the data corruption that happens without fencing.

### `Client`
Convenience wrapper that binds a `client_id` to a `LockService` and caches acquired tokens locally. `write_to_resource()` automatically attaches the correct fencing token to writes, so callers don't have to thread tokens manually.

## Patterns

- **Simulated time** — All time-dependent methods take `current_time` as a parameter rather than calling `time.time()`. This makes the entire system deterministic and testable without mocking.
- **Result dictionaries** — Write operations return `{'success': bool, 'error': str | None}` rather than raising exceptions. This models network responses where failure is a normal return value, not an exceptional condition.
- **No external dependencies** — Zero imports. Everything is in-memory, making this a pure reference implementation of the protocol.
- **Fenced vs unfenced comparison** — The two resource server classes share the same interface shape but differ on token validation, enabling direct A/B demonstration of why fencing matters.

## Dependencies

No imports. Imported by `test_fencing_tokens.py` and `tester_test_fencing_tokens.py` for testing.

## Flow

A typical fencing scenario plays out like this:

1. Client A calls `acquire_lock("db", time=0, ttl=10)` → gets token 1
2. Client A writes to `FencedResourceServer` with token 1 → accepted, highest becomes 1
3. Client A stalls (simulated by advancing time past TTL)
4. Client B calls `acquire_lock("db", time=15)` → lock expired, gets token 2
5. Client B writes with token 2 → accepted, highest becomes 2
6. Client A wakes up, tries to write with token 1 → **rejected** (1 < 2)

The `UnfencedResourceServer` path: step 6 would succeed, silently overwriting Client B's data.

## Invariants

1. **Token monotonicity** — `_counter` only increments. Every `acquire()` call that succeeds produces a strictly higher token than all previous ones. This is never reset.
2. **Fencing rejection** — `FencedResourceServer.write()` enforces `fencing_token >= highest`. Equal tokens are accepted (same holder writing multiple keys).
3. **Holder-only release/renew** — Both `release()` and `renew()` verify `client_id` matches the current holder. No other client can release or extend your lock.
4. **Renewal preserves token value** — `renew()` mutates `issued_at` and `ttl` on the existing `FencingToken` but does not change `token`. The fencing token number is stable across renewals.
5. **Re-entrant acquire issues new token** — If the same client re-acquires a lock it already holds, it gets a *new* (higher) token number. This differs from `renew()`.

## Error Handling

Errors are returned as values, never raised:
- `acquire()` returns `None` when the lock is held by another client
- `release()` returns `False` when the caller isn't the holder
- `renew()` returns `False` for expired, unheld, or wrong-client scenarios
- `FencedResourceServer.write()` returns `{'success': False, 'error': '...'}` for stale tokens
- `Client.write_to_resource()` returns `{'success': False, 'error': '...'}` when the client doesn't hold the named lock

No case throws an exception. This is consistent with modeling distributed system responses where "denied" is a normal outcome.

## Topics to Explore

- [file] `fencing-tokens/test_fencing_tokens.py` — See the race condition scenarios exercised: expired-lock takeover, stale-write rejection, fenced-vs-unfenced comparison
- [general] `ddia-ch8-process-pauses` — How GC pauses and network delays create the exact window fencing tokens protect against (DDIA §8.3)
- [function] `fencing-tokens/fencing_tokens.py:LockService.renew` — The distinction between renew (same token) and re-acquire (new token) has subtle correctness implications for long-running operations
- [file] `two-phase-commit/two_phase_commit.py` — Another distributed coordination primitive from the same book; compare how 2PC and fencing tokens address different failure modes
- [general] `redlock-controversy` — Martin Kleppmann's critique of Redis's Redlock algorithm centers on exactly the fencing token mechanism implemented here

## Beliefs

- `fencing-token-monotonic` — `LockService._counter` starts at 1, increments by 1 on every successful `acquire()`, and is never decremented or reset
- `fencing-rejects-stale-writes` — `FencedResourceServer.write()` rejects any write with a fencing token strictly less than the highest token previously seen for that resource
- `renew-preserves-token-number` — `LockService.renew()` extends the lock TTL without changing the `FencingToken.token` value, while `acquire()` by the same client issues a new higher token
- `no-exceptions-on-denial` — All denial conditions (lock held, stale token, wrong client) return sentinel values (`None`, `False`, or error dicts) rather than raising exceptions
- `unfenced-server-accepts-all` — `UnfencedResourceServer` has no token parameter on `write()` and accepts all writes unconditionally, serving as the unsafe baseline for comparison

