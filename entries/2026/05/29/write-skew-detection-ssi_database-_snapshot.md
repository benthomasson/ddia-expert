# Function: _snapshot in write-skew-detection/ssi_database.py

**Date:** 2026-05-29
**Time:** 10:00

## `SSIDatabase._snapshot`

### Purpose

`_snapshot` materializes the entire database state as it would appear to a reader at a specific point in time. It produces a point-in-time dictionary by scanning every key in the MVCC store and selecting the version visible at `snapshot_ts`. This is the mechanism that gives SSI transactions a consistent, frozen view of the database — the "snapshot" in Snapshot Isolation.

It exists to support two distinct needs:
1. **Predicate reads** (`read_predicate`) — the transaction needs the full key-value space to evaluate arbitrary predicate functions.
2. **Commit-time validation** — during `commit`, the database re-evaluates predicates against the *current* state and builds hypothetical snapshots for constraint checking.

### Contract

- **Precondition**: `snapshot_ts` must be a valid timestamp from the database's monotonic counter. No enforcement exists — passing an arbitrary integer won't raise, but will produce a semantically meaningless result.
- **Postcondition**: The returned dict contains exactly the keys that have a non-deleted, committed version with `commit_ts <= snapshot_ts`. Each key maps to the value from the *latest* such version.
- **Invariant**: The result is a *new* dict each call — callers can mutate it freely without affecting the store.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `snapshot_ts` | `int` | The logical timestamp defining the visibility horizon. Only versions committed at or before this timestamp are included. |

Edge case: if `snapshot_ts` is `0` or lower than any committed version, the result is an empty dict — the database "didn't exist yet."

### Return Value

A `dict[str, Any]` mapping keys to their visible values. Deleted keys and keys with no version at or before `snapshot_ts` are excluded (not present in the dict, not mapped to `None`). The caller gets a detached copy — the only obligation is understanding that this is a frozen view, not a live reference.

### Algorithm

1. Initialize an empty result dict.
2. Iterate over every key that has *ever* been written to the MVCC store (`self._store`).
3. For each key, delegate to `_visible_value(key, snapshot_ts)`, which scans all versions of that key and returns the value from the latest version with `commit_ts <= snapshot_ts`. If that version is a `_DELETED` sentinel, it returns `(None, writer_tx_id)`.
4. If the returned value is not `None` (meaning the key exists and isn't deleted at this timestamp), add it to the result dict.
5. Return the result.

The `_` in `val, _ = self._visible_value(...)` discards the writer transaction ID — `_snapshot` doesn't need causal tracking, only the values.

### Side Effects

None. This is a pure read operation — no mutations to `self._store`, no timestamp increments, no dependency graph updates.

### Error Handling

None. The method cannot raise under normal use. `_visible_value` handles missing keys gracefully by returning `(None, None)`. If `self._store` contains malformed version tuples, the unpacking in `_visible_value` would raise a `ValueError`, but that indicates a corrupted store, not a recoverable error.

### Usage Patterns

Three call sites in `commit` and `read_predicate`:

1. **`read_predicate`** — called with `tx.start_timestamp` to build the transaction's consistent view before evaluating the predicate function. The result is merged with buffered writes/deletes to reflect uncommitted local changes.

2. **Phantom detection in `commit`** — called with `self._next_timestamp` (the *current* logical time, strictly greater than all committed timestamps) to get the latest committed state. This is compared against the predicate's original result to detect phantoms.

3. **Constraint validation in `commit`** — called with `self._next_timestamp` again, then mutated in-place with the transaction's buffered writes and deletes to build a hypothetical post-commit state for constraint checking.

Caller obligation: understand that this snapshot does **not** include uncommitted writes from any active transaction. If you need the transaction-local view, you must overlay `tx._writes` and `tx._deletes` yourself (as `read_predicate` and constraint validation both do).

### Dependencies

- **`self._store`** — the MVCC version store, structured as `dict[key, list[tuple[commit_ts, value, tx_id]]]`.
- **`self._visible_value`** — performs the per-key version resolution with linear scan over the version chain.
- **`_DELETED` sentinel** — `_visible_value` converts deleted keys to `None` values, so `_snapshot` never needs to handle the sentinel directly.

### Performance Note

This is an O(K × V) operation where K is the number of distinct keys and V is the average number of versions per key, since `_visible_value` does a linear scan over all versions. For a production system this would be a concern, but for a reference implementation demonstrating SSI concepts, correctness over performance is the right trade-off.

## Topics to Explore

- [function] `write-skew-detection/ssi_database.py:_visible_value` — The per-key MVCC version resolution that `_snapshot` delegates to; understanding its linear scan and deletion handling is essential
- [function] `write-skew-detection/ssi_database.py:read_predicate` — The primary consumer of `_snapshot`, showing how predicate locks and phantom detection work together
- [function] `write-skew-detection/ssi_database.py:commit` — Where `_snapshot` is used at commit time for both phantom re-evaluation and constraint validation against hypothetical state
- [general] `snapshot-isolation-vs-ssi` — How plain Snapshot Isolation allows write skew, and why SSI adds read-write conflict detection and predicate locks on top
- [general] `mvcc-version-chain-design` — Alternative version chain structures (linked lists, index-organized) and how they affect snapshot materialization cost

## Beliefs

- `snapshot-returns-detached-copy` — `_snapshot` returns a new dict each call; callers in `read_predicate` and `commit` mutate it freely without corrupting the MVCC store
- `snapshot-excludes-uncommitted-writes` — `_snapshot` only reflects committed versions; all three call sites manually overlay `tx._writes` and `tx._deletes` when they need the transaction-local view
- `snapshot-uses-next-timestamp-for-current-state` — During commit validation, `_snapshot(self._next_timestamp)` is used to capture all committed versions because `_next_timestamp` is strictly greater than any existing `commit_ts`
- `snapshot-discards-writer-identity` — `_snapshot` drops the writer `tx_id` returned by `_visible_value`; causal dependency tracking is handled elsewhere (in `read` and `read_predicate`)

