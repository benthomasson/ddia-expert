# Function: get in read-repair/read_repair.py

**Date:** 2026-05-29
**Time:** 13:38



## `Replica.get` — Versioned Key-Value Lookup

### Purpose

`get` is the read path for a single replica node in a leaderless replication system. It retrieves the current value and its version number for a given key. This method is the low-level building block that higher-level quorum reads (`ReadRepairStore.get`) and anti-entropy repair (`anti_entropy_repair`) call repeatedly across multiple replicas to compare versions and detect staleness.

### Contract

- **Precondition**: `key` must be a string. No enforcement beyond Python's duck typing.
- **Postcondition**: Returns the exact tuple stored by `put`, or `None`. Never mutates state.
- **Invariant**: The returned tuple, if non-`None`, always has the shape `(value, version: int)` — this is established by `put` and assumed by every caller.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `str` | The lookup key. No validation — any string is accepted, including empty strings. |

### Return Value

- **`(value, version)`** — A 2-tuple where `value` is the stored data (any type) and `version` is an `int` monotonically assigned by `ReadRepairStore.put`. Returned when the key exists in this replica's store.
- **`None`** — Returned when the key has never been written to this replica. This is the critical signal: callers use `None` to distinguish "missing" from "present at version 0" (version 0 is never actually written since `ReadRepairStore.put` starts versions at 1).

Every caller in this file unpacks the result with a `None` guard first:

```python
result = r.get(key)
if result is not None:
    val, ver = result
```

The caller must handle `None` before attempting to destructure — there is no sentinel tuple.

### Algorithm

1. Check if `key` exists in the `_store` dictionary via `in`.
2. If present, return the stored `(value, version)` tuple directly (no copy).
3. If absent, return `None`.

This is a direct dictionary lookup — O(1) average case. No version filtering, no tombstone handling, no conflict resolution. All of that complexity lives in `ReadRepairStore`.

### Side Effects

None. This is a pure read with no mutations, no I/O, and no logging. It does not check `self._available` — availability filtering is the caller's responsibility (`ReadRepairStore._available_replicas()`).

### Error Handling

No exceptions are raised or caught. The method relies on Python's `dict.__contains__` and `dict.__getitem__`, neither of which can fail for a valid key type. If a non-hashable key were passed, Python would raise `TypeError` from the dict lookup — this is not caught.

### Usage Patterns

`get` is never called in isolation by external code. It's always called by `ReadRepairStore` methods in one of three patterns:

1. **Version discovery during writes** (`ReadRepairStore.put`): reads from *all* replicas (including unavailable ones) to find the max version before incrementing.
2. **Quorum reads** (`ReadRepairStore.get`): reads from R available replicas, compares versions, and triggers repair on stale ones.
3. **Anti-entropy / state inspection** (`anti_entropy_repair`, `get_replica_states`): reads from available replicas to find and fix divergence.

### Dependencies

- `self._store`: a plain `dict` — no external modules, no persistence, no serialization. The entire store is in-memory and lost on process exit.

---

## Topics to Explore

- [function] `read-repair/read_repair.py:ReadRepairStore.get` — The quorum read that calls `Replica.get` across multiple nodes and performs read repair on stale responses
- [function] `read-repair/read_repair.py:Replica.put` — The write counterpart; its version-gating logic (`version < current_version`) defines what "stale" means
- [function] `read-repair/read_repair.py:ReadRepairStore.anti_entropy_repair` — Background convergence mechanism that complements read-path repair
- [file] `hinted-handoff/hinted_handoff.py` — Alternative consistency recovery strategy for when replicas are temporarily unavailable
- [general] `quorum-consistency-guarantees` — How R + W > N ensures strong consistency and what happens when that condition isn't met (as warned in `__init__`)

---

## Beliefs

- `replica-get-returns-tuple-or-none` — `Replica.get` returns a `(value, version)` tuple when the key exists, or `None` when it doesn't; there is no sentinel version for "missing"
- `replica-get-has-no-side-effects` — `Replica.get` performs no mutations, availability checks, or I/O — it is a pure dictionary lookup
- `replica-get-does-not-check-availability` — `Replica.get` does not consult `_available`; callers are responsible for filtering unavailable replicas before calling
- `replica-versions-start-at-one` — Versions assigned by `ReadRepairStore.put` start at 1, so a version of 0 in return values from `ReadRepairStore.get` means "key not found," not "initial version"

