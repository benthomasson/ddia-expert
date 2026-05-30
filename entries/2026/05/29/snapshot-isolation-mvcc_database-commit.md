# Function: commit in snapshot-isolation/mvcc_database.py

**Date:** 2026-05-29
**Time:** 10:07

## `MVCCDatabase.commit` — Transaction Commit with Write-Write Conflict Detection

### Purpose

`commit` finalizes a transaction, making its writes durable and visible to future transactions. Before committing a read-write transaction, it performs **first-committer-wins** conflict detection: if another transaction wrote to any of the same keys and committed during this transaction's lifetime, the current transaction is aborted instead. This is the core mechanism that enforces **snapshot isolation** — each transaction operates on a consistent snapshot, and concurrent writes to the same key are not allowed to silently overwrite each other.

### Contract

**Preconditions:**
- `tx` must be an active transaction (status `"active"`). Calling commit on an already-committed or aborted transaction raises `TransactionError`.

**Postconditions (on success):**
- `tx._status` is `"committed"`
- `tx.tx_id` is in `self._committed`
- A commit timestamp is assigned and stored in `self._commit_timestamps`

**Postconditions (on conflict):**
- `tx` is aborted via `self.abort(tx)` — status becomes `"aborted"`, write-set side effects (deletions) are rolled back
- Returns `False`

**Invariant:** At most one transaction that wrote to a given key can commit per conflict window. This prevents lost updates.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `tx` | `Transaction` | The transaction to commit. Must have been returned by `begin_transaction`. |

### Return Value

Returns `bool`:
- `True` — commit succeeded, writes are now visible to future snapshots
- `False` — write-write conflict detected, transaction was aborted

The caller must check this return value. After `False`, the transaction is dead — any further operations on it will raise `TransactionError`.

### Algorithm

1. **Guard:** Verify the transaction is still active.

2. **Fast path for read-only transactions:** If `tx.read_only`, skip conflict detection entirely — read-only transactions can never conflict. Mark committed immediately.

3. **Write-write conflict detection:** For each key in the transaction's write set, scan all versions of that key looking for a conflicting version. A version `v` is conflicting if all of the following hold:

   - `v.created_by != tx.tx_id` — it was written by a *different* transaction
   - `v.created_by not in self._aborted` — that transaction wasn't aborted
   - `v.created_by in self._committed` — that transaction has committed
   - `v.created_by in tx.active_at_start or v.created_by >= tx.tx_id` — the creating transaction was either **still active when we started** (meaning it committed *during* our lifetime) or it **started after us** (meaning it's entirely concurrent)

   The last condition is the key insight: if a transaction committed *before* we started and wasn't in our active set, its write is part of our snapshot — that's not a conflict. A conflict only exists when a concurrent transaction committed a write to the same key.

   On conflict, the method calls `self.abort(tx)` and returns `False`.

4. **Commit:** Set status to `"committed"`, add to the committed set, allocate a monotonically increasing commit timestamp, and store it.

### Side Effects

- **Mutates `tx._status`** — sets to `"committed"` or (via `abort`) `"aborted"`
- **Mutates `self._committed`** — adds the transaction ID
- **Mutates `self._commit_timestamps`** — records the commit timestamp
- **Increments `self._next_timestamp`** — allocates a commit timestamp
- **On conflict, calls `self.abort(tx)`** — which also rolls back deletion markers in the version chains

### Error Handling

- Raises `TransactionError` if the transaction is not active (already committed or aborted)
- Write-write conflicts are **not** raised as exceptions — they return `False` and abort silently. This is a design choice: conflicts are expected in concurrent workloads and are part of normal control flow.

### Usage Patterns

```python
db = MVCCDatabase()
tx = db.begin_transaction()
db.write(tx, "account:1", 500)

if not db.commit(tx):
    # Conflict — retry with a new transaction
    tx = db.begin_transaction()
    # ... re-read and re-apply logic
```

Callers are expected to implement retry loops around commit failures. The abort-on-conflict pattern means the caller never needs to manually call `abort` after a failed commit.

### Dependencies

- `self.abort(tx)` — delegates rollback on conflict
- `self._check_active(tx)` — precondition guard
- `Transaction.write_set` — the set of keys written, populated by `write()` and `delete()`
- `Transaction.active_at_start` — snapshot of active transaction IDs at begin time, set by `begin_transaction()`
- `self._versions`, `self._committed`, `self._aborted` — global MVCC state

### Notable Assumptions

1. **Single-threaded execution.** There's no locking — the conflict check and commit are not atomic. In a multi-threaded environment, two transactions could both pass the conflict check before either commits, violating first-committer-wins.

2. **The conflict check scans *all* versions of each key**, not just the latest. This is O(versions) per key in the write set, which could degrade if versions accumulate without garbage collection.

3. **The conflict condition `v.created_by >= tx.tx_id` catches transactions that started after us**, but this relies on monotonically increasing transaction IDs correlating with temporal ordering — which holds because `begin_transaction` increments `_next_tx_id` sequentially.

4. **Commit timestamps are separate from transaction IDs and start timestamps.** A transaction gets a start timestamp at begin time and a different commit timestamp at commit time. This two-timestamp design supports the visibility rules in `_is_visible`, though notably the commit timestamp isn't actually used by `_is_visible` — visibility is determined purely by transaction IDs and the committed/active sets.

---

## Topics to Explore

- [function] `snapshot-isolation/mvcc_database.py:_is_visible` — The visibility function that determines which versions a transaction can see; the dual of the conflict check in `commit`
- [function] `snapshot-isolation/mvcc_database.py:abort` — How rollback works, specifically the deletion marker cleanup that `commit` triggers on conflict
- [file] `snapshot-isolation/test_mvcc.py` — Test cases that exercise concurrent write scenarios and conflict detection edge cases
- [general] `write-skew-anomaly` — Snapshot isolation prevents lost updates (via this conflict check) but still allows write skew — explore how SSI extends this
- [function] `snapshot-isolation/mvcc_database.py:garbage_collect` — How old versions are reclaimed, relevant because the conflict check scans all versions per key

---

## Beliefs

- `commit-first-committer-wins` — Write-write conflicts are resolved by first-committer-wins: if two transactions write to the same key, the one that commits first succeeds and the second is aborted
- `commit-read-only-skip-conflict-check` — Read-only transactions bypass conflict detection entirely and always commit successfully
- `commit-timestamp-unused-by-visibility` — Commit timestamps are assigned and stored but never referenced by `_is_visible`; visibility uses transaction IDs and the committed/active-at-start sets instead
- `commit-aborts-on-conflict` — On conflict, `commit` calls `abort` internally before returning `False`, so the caller never needs to manually abort after a failed commit
- `commit-single-threaded-assumption` — The conflict-check-then-commit sequence is not atomic; correct behavior assumes single-threaded execution with no concurrent calls to `commit`

