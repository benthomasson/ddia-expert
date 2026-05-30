# Function: put in read-repair/read_repair.py

**Date:** 2026-05-29
**Time:** 14:05

## `Replica.put` — Versioned Write with Staleness Guard

### Purpose

`put` is the write primitive for a single replica node in a leaderless replication system. It stores a key-value pair **only if the incoming version is not older than what the replica already holds**. This prevents read-repair and anti-entropy processes from accidentally overwriting newer data with stale values — a critical safety property in eventually consistent systems.

### Contract

- **Precondition**: `version` should be a non-negative integer. The caller is responsible for determining the correct version (typically `max(all replicas) + 1` for new writes, or the winning version during repair).
- **Postcondition**: After a successful `put`, `self._store[key] == (value, version)`. After a rejected `put`, the store is unchanged.
- **Invariant**: The version stored for any key is monotonically non-decreasing — it can increase or stay the same, but never go backward.

### Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `key` | `str` | The logical key to store. No validation — empty strings are accepted. |
| `value` | any | The payload. Untyped; the store treats it as opaque. |
| `version` | `int` | A logical timestamp. Compared against the current version to decide whether to accept the write. |

**Edge cases**: If `version == current_version`, the write is accepted and the value is overwritten. This means two writes at the same version are last-writer-wins with no conflict detection.

### Return Value

Returns `bool`:
- `True` — the write was accepted and the store was mutated.
- `False` — the write was rejected because the replica already holds a strictly newer version.

The caller can use this to detect stale writes but in practice the return value is ignored by all callers in this module (`ReadRepairStore.put`, read repair in `get`, and `anti_entropy_repair` all call `r.put()` without checking the result).

### Algorithm

1. Check if `key` already exists in `_store`.
2. If it does, extract the current version and compare: if `version < current_version`, reject with `False`.
3. Otherwise (key is new, or `version >= current_version`), overwrite `_store[key]` with `(value, version)` and return `True`.

The comparison is **strictly less than** — equal versions are accepted. This is a deliberate choice: during read repair, the winning version is pushed to stale replicas, and some replicas may already hold that exact version. Accepting `>=` makes repair idempotent for same-version writes.

### Side Effects

Mutates `self._store` on success. No I/O, no logging, no callbacks. This is a pure in-memory operation.

### Error Handling

None. No exceptions are raised. Invalid types for `version` (e.g., passing a string) would produce a `TypeError` at the `<` comparison, but this is not guarded.

### Usage Patterns

`put` is called in three contexts within `ReadRepairStore`:

1. **Normal writes** (`ReadRepairStore.put`, line ~68): The store computes `max_version + 1` across all replicas, then calls `r.put(key, value, new_version)` on the first W available replicas. Since the version is freshly computed, the guard never triggers here.

2. **Read repair** (`ReadRepairStore.get`, line ~105): After a quorum read discovers stale replicas, it pushes the winning `(value, version)` to them. The guard protects against the (unlikely in this implementation) case where a concurrent write advanced the version between the read and the repair.

3. **Anti-entropy repair** (`anti_entropy_repair`, line ~124): Same pattern — find the max version, push it to all replicas with older data.

### Dependencies

None beyond Python builtins. The method operates entirely on `self._store`, a plain `dict`.

### Assumptions Not Enforced by Types

- `version` is assumed to be a comparable integer. Passing a float, `None`, or a string would break silently or raise at runtime.
- Version numbering is assumed to be externally coordinated. `put` does not generate versions — it trusts the caller. If two callers independently assign the same version to different values, the last write wins silently.
- There is no concurrency control. In a threaded environment, the check-then-write on lines 22–25 is a race condition (TOCTOU).

---

## Topics to Explore

- [function] `read-repair/read_repair.py:ReadRepairStore.get` — The read-repair logic that calls `put` to heal stale replicas during quorum reads
- [function] `read-repair/read_repair.py:ReadRepairStore.put` — The store-level write that coordinates version assignment across replicas before calling `Replica.put`
- [function] `read-repair/read_repair.py:anti_entropy_repair` — Background repair mechanism that uses `put` to sync all replicas, complementing the read-path repair
- [file] `read-repair/test_read_repair.py` — Test cases that exercise version conflict and repair scenarios
- [general] `quorum-consistency-guarantees` — How R + W > N ensures a read quorum always overlaps with the write quorum, and what breaks when it doesn't

## Beliefs

- `replica-put-rejects-stale-versions` — `Replica.put` rejects writes where `version < current_version` but accepts `version == current_version`, making same-version repair idempotent
- `replica-put-return-value-unused` — No caller in `ReadRepairStore` checks the `bool` return value of `Replica.put`; rejected writes are silently ignored
- `replica-put-no-concurrency-safety` — The version check and store update in `Replica.put` are not atomic; the method is unsafe under concurrent access without external locking
- `replica-versions-caller-coordinated` — `Replica.put` does not generate version numbers; version assignment is the responsibility of `ReadRepairStore.put`, which reads the max across all replicas before writing

