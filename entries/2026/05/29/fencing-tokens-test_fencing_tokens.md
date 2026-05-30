# File: fencing-tokens/test_fencing_tokens.py

**Date:** 2026-05-29
**Time:** 13:02

I'll work from the test file content you provided, which gives a clear picture of the implementation's API surface.

---

# `fencing-tokens/test_fencing_tokens.py`

## Purpose

This is the test suite for a **fencing tokens** implementation — a distributed systems primitive from DDIA Chapter 8 (The Trouble with Distributed Systems). Fencing tokens solve the problem where a client holds a lock, pauses (e.g., GC pause), the lock expires, another client acquires it, and the original client wakes up believing it still holds the lock. Without fencing, the stale client can corrupt shared state.

The file's responsibility is to verify all components of the fencing token protocol: lock lifecycle, token monotonicity, server-side token enforcement, and the contrast between safe (fenced) and unsafe (unfenced) resource access.

## Key Components

The tests exercise five classes imported from `fencing_tokens.py`:

| Class | Role |
|-------|------|
| `FencingToken` | Value object — carries `token` (int), `lock_name`, `client_id` |
| `LockService` | Central lock manager — issues monotonically increasing tokens, enforces mutual exclusion, handles TTL expiry and renewal |
| `FencedResourceServer` | Storage server that **rejects** writes carrying a token lower than the highest it has seen per resource |
| `UnfencedResourceServer` | Storage server with **no token checking** — accepts all writes, used to demonstrate the unsafe scenario |
| `Client` | Wraps `LockService` and `FencedResourceServer` interactions — holds acquired tokens and attaches them to writes |

### `LockService` API

- `acquire(lock_name, client_id, current_time, ttl) → FencingToken | None` — returns `None` if lock is held by another client
- `release(lock_name, client_id) → bool`
- `renew(lock_name, client_id, current_time, ttl) → bool` — extends TTL without changing the token value
- `is_held(lock_name, current_time) → dict | None` — returns `{'client_id': ...}` if held
- `get_token_counter() → int` — exposes the global counter (next value to issue)

### `FencedResourceServer` API

- `write(resource, key, value, fencing_token) → dict` — returns `{'success': True/False}`
- `read(resource, key) → value`

### `Client` API

- `acquire_lock(lock_name, current_time, ttl) → FencingToken | None`
- `release_lock(lock_name)`
- `write_to_resource(server, resource, key, value, lock_name) → dict` — attaches the stored token for `lock_name`; returns `{'success': False, 'error': '...does not hold...'}` if the client has no token for that lock

## Patterns

**Simulated time.** The `LockService` takes `current_time` as an explicit parameter instead of reading a clock. This makes tests deterministic and lets tests advance time arbitrarily (e.g., jumping from time 0 to time 11 to expire a TTL=10 lock) without sleeps or mocking.

**Contract-first test organization.** The five test classes map to distinct behavioral contracts:

| Class | What it covers |
|-------|---------------|
| `TestLockAcquisition` | Core lock lifecycle: acquire, mutual exclusion, expiry, release, token uniqueness |
| `TestLockRenewal` | Renewal extends TTL without incrementing the global counter or changing the token |
| `TestFencedServer` | Server-side token enforcement: accept higher, reject lower, per-resource tracking |
| `TestScenarios` | End-to-end scenarios contrasting safe vs. unsafe behavior |
| `TestClientEdgeCases` | Error paths: writing without a lock, concurrent contention, token validity after release |

**Unsafe-by-design demonstration.** `test_unsafe_scenario_stale_write_corrupts` (test 9) is intentionally a "bad outcome" test — it asserts that corruption *does* happen without fencing. This is pedagogical: it demonstrates the problem before `test_safe_scenario_stale_write_rejected` (test 10) demonstrates the solution.

## Dependencies

**Imports:**
- `pytest` — test framework (imported but no fixtures or parametrize decorators are used; tests are plain method-based)
- `fencing_tokens` — the implementation module; imports all five public classes

**Imported by:**
- `tester_test_fencing_tokens.py` — likely a meta-test or test runner wrapper (consistent with the project's `tester_test_*.py` convention seen across all modules)

## Flow

The tests follow a recurring pattern:

1. **Create a `LockService`** — fresh per test, so no shared state leaks between tests
2. **Acquire locks** with explicit timestamps to control timing
3. **Advance time** by passing a later `current_time` to simulate expiry
4. **Perform writes** through either a `Client` (which attaches the token) or directly on the server (to test raw token enforcement)
5. **Assert outcomes** — token values, success/failure flags, stored data, error messages

The two scenario tests (9 and 10) follow a full lifecycle: two clients, one lock, expiry in the middle, stale write at the end. The difference is the resource server type.

## Invariants

1. **Global monotonicity** — every `acquire` call that succeeds returns a token strictly greater than all previously issued tokens, regardless of which lock or client is involved (test 2)
2. **Mutual exclusion** — at most one client holds a given lock at any time; a second `acquire` returns `None` (test 3)
3. **TTL-based expiry** — a lock acquired at time `t` with TTL `d` is available at time `t + d` (test 4, boundary at exact expiry time)
4. **Renewal preserves token identity** — `renew` extends the TTL but does not change the token value or increment the global counter (test 6)
5. **Server-side token ordering** — `FencedResourceServer` rejects any write whose `fencing_token` is less than the highest token previously accepted for that resource (test 8)
6. **Per-resource independence** — token tracking on the server is scoped per resource, not global; a lower token can succeed on a different resource (test 11)
7. **Token uniqueness** — every successful `acquire` produces a distinct token value, even for the same lock and client (test 15)
8. **Tokens don't expire at the server** — a token remains valid at the `FencedResourceServer` even after the lock that issued it has been released (test 14). The server cares about ordering, not lock state.

## Error Handling

- **Lock contention:** `acquire` returns `None` (not an exception) when the lock is held — the caller checks for it
- **Write without lock:** `Client.write_to_resource` returns `{'success': False, 'error': '...does not hold...'}` — it does not raise
- **Stale token rejection:** `FencedResourceServer.write` returns `{'success': False}` — the data is unchanged, and the caller inspects the dict
- All error paths use **return-value signaling** (dicts with success/error keys), not exceptions. This is consistent with distributed systems APIs where failures are expected conditions, not exceptional ones.

## Topics to Explore

- [file] `fencing-tokens/fencing_tokens.py` — The implementation: how the global counter works, how TTL expiry is checked, and how `FencedResourceServer` tracks high-water marks per resource
- [function] `fencing-tokens/fencing_tokens.py:LockService.renew` — Renewal is a subtle operation: it must extend TTL without touching the counter, which means the counter tracks acquisitions, not interactions
- [general] `fencing-vs-epoch-based-locking` — DDIA discusses fencing tokens alongside epoch numbers in consensus protocols (e.g., Raft term numbers) — worth comparing the two monotonic ordering mechanisms
- [file] `fencing-tokens/tester_test_fencing_tokens.py` — The meta-test file likely validates test quality or runs the tests through a mutation/property-based harness
- [general] `simulated-time-pattern` — The explicit `current_time` parameter pattern appears across several modules in this repo (leader-election, hinted-handoff); understanding this pattern is key to reading any of the distributed systems test suites

## Beliefs

- `fencing-token-counter-is-global` — The token counter is global across all locks, not per-lock; tokens from different locks are comparable and ordered
- `fenced-server-rejects-by-high-water-mark` — `FencedResourceServer` tracks the highest token seen per resource and rejects any write with a lower token, making stale writes impossible
- `fencing-token-tracking-is-per-resource` — Token high-water marks on the server are scoped per resource identifier, so independent resources don't interfere with each other's ordering
- `lock-renewal-does-not-increment-counter` — Renewing a lock extends its TTL but does not change the token value or advance the global counter, preserving the invariant that only new acquisitions produce new tokens
- `fencing-errors-use-return-values-not-exceptions` — All failure conditions (contention, stale tokens, missing locks) are signaled via return values (`None` or `{'success': False}`), not exceptions

