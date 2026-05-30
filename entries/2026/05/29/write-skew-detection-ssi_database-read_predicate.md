# Function: read_predicate in write-skew-detection/ssi_database.py

**Date:** 2026-05-29
**Time:** 10:11

# `SSIDatabase.read_predicate`

## Purpose

`read_predicate` performs a predicate-based range read under snapshot isolation, returning all key-value pairs that satisfy an arbitrary predicate function. Its critical secondary purpose is recording the predicate and its result set so the database can later detect **phantoms** at commit time — rows that were inserted, deleted, or modified by a concurrent transaction such that the predicate would now return different results. Without this, SSI would miss write skew anomalies that operate through set membership changes rather than individual key modifications.

This is the mechanism behind what the literature calls **predicate locks** (or **index-range locks** in practical implementations). A plain `read()` tracks individual keys, but that's insufficient when a transaction's correctness depends on the *absence* of certain rows (e.g., "no other doctor is on call"). `read_predicate` closes that gap.

## Contract

**Preconditions:**
- `tx` must be an active `SSITransaction` (`tx._status == "active"`). Calling on an aborted or committed transaction raises `RuntimeError`.
- `predicate` must be a callable accepting `(key, value)` and returning a truthy/falsy value. Exceptions from the predicate are silently swallowed (the key is skipped).

**Postconditions:**
- Every key in the returned dict has been added to `tx._read_set`.
- A `(predicate, result_snapshot)` tuple has been appended to `tx._predicate_locks`.
- For every returned key that existed in the committed snapshot (not just in the transaction's local buffer), a causal dependency edge has been recorded in `self._dependency_graph`.
- If `self._pessimistic` is true, any read conflict with a concurrent committed transaction has already triggered an abort.

**Invariant:** The result reflects the transaction's consistent snapshot merged with its own uncommitted writes and deletes — the "read-your-own-writes" guarantee.

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `tx` | `SSITransaction` | The active transaction performing the read. Provides the snapshot timestamp, local write/delete buffers, and the read set to update. |
| `predicate` | `Callable[[key, value], bool]` | An arbitrary filter function. Evaluated against every key-value pair visible to the transaction. Any exception it raises causes that pair to be silently skipped. |

**Edge cases:**
- A predicate that matches nothing returns `{}` but still records a predicate lock — this is correct because at commit time the system must verify that nothing *new* matches either.
- A predicate that throws on every pair returns `{}` with no keys in the read set, but the predicate lock is still stored.

## Return Value

Returns a `dict[key, value]` of all matching pairs. The caller receives a snapshot-consistent view that includes the transaction's own buffered writes. The caller does not need to handle errors from the return value itself — conflicts surface later at `commit()` or immediately via `RuntimeError` in pessimistic mode.

## Algorithm

**Step 1 — Build the merged snapshot:**
```python
snap = self._snapshot(tx.start_timestamp)
merged = dict(snap)
```
Constructs the MVCC snapshot at the transaction's start timestamp (all versions with `commit_ts <= start_timestamp`). Then overlays the transaction's own buffered writes and removes its pending deletes. This gives the transaction a "read-your-own-writes" view.

**Step 2 — Evaluate the predicate:**
```python
for k, v in merged.items():
    if predicate(k, v):
        result[k] = v
        tx._read_set.add(k)
```
Applies the predicate to every key-value pair in the merged view. Matching keys are added to the read set (so standard read-write conflict detection also covers them). Predicate exceptions are caught and the key is skipped — this is a design choice to make predicates lenient with heterogeneous value types.

**Step 3 — Record causal dependencies:**
```python
for k in result:
    if k in snap:
        _, writer_tx_id = self._visible_value(k, tx.start_timestamp)
        if writer_tx_id is not None:
            self._dependency_graph.setdefault(tx.tx_id, set()).add(writer_tx_id)
```
For each matching key that came from committed data (not from the transaction's own buffer), records a dependency edge from this transaction to the transaction that wrote that value. This builds the serialization graph used by SSI. The `if k in snap` guard avoids recording dependencies for keys the transaction itself created.

**Step 4 — Store the predicate lock:**
```python
tx._predicate_locks.append((predicate, dict(result)))
```
Saves the predicate function and a copy of the current result set. At commit time, `commit()` re-evaluates each predicate against the then-current database state and compares keys and values — any difference is a phantom conflict.

**Step 5 — Pessimistic conflict check (optional):**
```python
if self._pessimistic:
    for k in result:
        self._check_read_conflict(tx, key)
```
In pessimistic mode, immediately checks whether any concurrent committed transaction has written to any of the matched keys. If so, the transaction is aborted with a `RuntimeError` on the spot rather than waiting until commit.

## Side Effects

1. **Mutates `tx._read_set`** — adds all matching keys.
2. **Mutates `tx._predicate_locks`** — appends the `(predicate, result)` tuple.
3. **Mutates `self._dependency_graph`** — adds causal edges for committed data that was read.
4. **May mutate `tx._status`** — in pessimistic mode, `_check_read_conflict` sets status to `"aborted"`.
5. **No I/O** — all state is in-memory.

## Error Handling

| Condition | Behavior |
|-----------|----------|
| `tx._status != "active"` | Raises `RuntimeError` with a descriptive message. |
| Predicate throws on a particular `(k, v)` | Caught by bare `except Exception: pass` — the key is silently excluded from results. This means a buggy predicate degrades to returning fewer results rather than crashing. |
| Pessimistic read conflict | `_check_read_conflict` raises `RuntimeError` and sets `tx._status = "aborted"`. The transaction is dead after this. |

**Notable:** predicate exceptions are swallowed without logging, which could mask bugs in caller-provided predicates.

## Usage Patterns

A typical use is checking an invariant over a set of rows before making a conflicting write — the classic write skew scenario:

```python
tx = db.begin_transaction()

# "How many doctors are on call?"
on_call = db.read_predicate(tx, lambda k, v: v.get("on_call") is True)

if len(on_call) >= 2:
    # Safe to take one off call
    db.write(tx, "doctor:alice", {"on_call": False})

result = db.commit(tx)
# If a concurrent tx also took a doctor off call and committed first,
# the predicate lock detects the phantom and aborts this tx.
```

**Caller obligations:**
- The predicate must be a pure function of `(key, value)` — it shouldn't close over mutable external state, since it will be re-evaluated at commit time and must produce consistent results.
- Callers must check the commit result; the predicate lock only triggers conflict detection, not prevention (unless pessimistic mode is on).

## Dependencies

- **`_snapshot(timestamp)`** — builds the base MVCC snapshot.
- **`_visible_value(key, timestamp)`** — retrieves the specific version visible at a timestamp, including the writer's transaction ID for dependency tracking.
- **`_dependency_graph`** — the shared serialization graph that `commit()` could use for cycle detection (though this implementation uses direct conflict checks rather than full cycle detection).
- **`_check_read_conflict(tx, key)`** — pessimistic-mode early abort check.
- **`SSITransaction`** — the transaction object whose `_read_set`, `_writes`, `_deletes`, `_predicate_locks`, and `_status` fields are all directly accessed.

## Topics to Explore

- [function] `write-skew-detection/ssi_database.py:commit` — Where predicate locks are re-evaluated against current state to detect phantoms; the other half of this mechanism
- [function] `write-skew-detection/ssi_database.py:_visible_value` — The MVCC version selection logic that determines which value a snapshot sees and which transaction wrote it
- [file] `write-skew-detection/test_ssi.py` — Test cases that exercise predicate-based phantom detection, showing the write skew scenarios this method is designed to catch
- [general] `predicate-locks-vs-index-range-locks` — How real databases (PostgreSQL SSI, InnoDB gap locks) approximate predicate locks with coarser-grained mechanisms for performance
- [function] `write-skew-detection/ssi_database.py:read` — The single-key read path, which tracks individual keys but cannot detect phantom-based write skew

## Beliefs

- `read-predicate-stores-result-snapshot` — `read_predicate` stores a deep copy of its result dict alongside the predicate function, so `commit()` can compare against the original result even after the database state has changed
- `read-predicate-includes-own-writes` — The merged snapshot includes the transaction's own buffered writes and excludes its pending deletes, providing read-your-own-writes semantics within predicate evaluation
- `predicate-exceptions-silently-skipped` — If the predicate function throws on a given key-value pair, that pair is silently excluded from results with no logging or error propagation
- `read-predicate-causal-deps-only-for-committed-data` — Dependency graph edges are only recorded for keys that exist in the committed snapshot (`k in snap`), not for keys the transaction itself buffered via `write()`
- `empty-predicate-match-still-locks` — A predicate that matches zero keys still appends a predicate lock, enabling phantom detection if a concurrent transaction later inserts a matching key

