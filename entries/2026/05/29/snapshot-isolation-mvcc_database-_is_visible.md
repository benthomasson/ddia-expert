# Function: _is_visible in snapshot-isolation/mvcc_database.py

**Date:** 2026-05-29
**Time:** 09:58

# `_is_visible(self, tx, version)` — MVCC Snapshot Visibility Check

## Purpose

This is the core visibility function of the snapshot isolation implementation. It answers: **"Should transaction `tx` be able to see this particular version of a record?"**

In MVCC (Multi-Version Concurrency Control), every write creates a new version rather than overwriting in place. Multiple transactions can be in flight simultaneously, each seeing a consistent snapshot of the database as it existed when they started. `_is_visible` enforces this illusion by filtering which versions each transaction can observe.

Without this function, transactions would see uncommitted writes from other transactions (dirty reads), partially-committed state, or data that was deleted after their snapshot was taken.

## Contract

**Preconditions:**
- `tx` is a `Transaction` object with a valid `tx_id` and populated `active_at_start` set
- `version` is a `Version` object with `created_by` set (and `deleted_by` optionally set)
- `self._committed` and `self._aborted` accurately reflect the current state of all transactions

**Postconditions:**
- Returns `True` if `version` should appear in `tx`'s snapshot, `False` otherwise
- No state is mutated

**Invariant:** A version is visible if and only if:
1. Its creating transaction is visible to `tx`, AND
2. It has not been deleted by a transaction visible to `tx`

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `tx` | `Transaction` | The observing transaction. Its `tx_id`, `active_at_start` set determine what snapshot it sees. |
| `version` | `Version` | The candidate version being checked. Has `created_by` and `deleted_by` fields referencing transaction IDs. |

**Edge cases:**
- `version.deleted_by` can be `None` (not deleted)
- `tx` checking its own writes (`created_by == tx.tx_id`)
- `tx` checking a version deleted by itself (`deleted_by == tx.tx_id`)

## Return Value

`bool` — `True` if the version is visible to `tx`, `False` otherwise.

The caller (`read`, `delete`, `scan`) uses this to filter the version list. If multiple versions of the same key are visible, the caller takes the last one (latest visible).

## Algorithm

The function has two phases: **creation visibility** and **deletion visibility**.

### Phase 1: Is the creating transaction visible?

```
created_by = version.created_by
```

**Step 1 — Aborted check:** If the creator was aborted, return `False` immediately. Aborted transactions' effects must never be seen by anyone.

**Step 2 — Self-write check:** If `created_by == tx.tx_id`, the transaction sees its own writes. This is essential — without it, a transaction couldn't read back what it just wrote.

**Step 3 — Committed-and-in-snapshot check:** If the creator committed, it's visible only if ALL three conditions hold:
1. `created_by in self._committed` — the creator actually committed
2. `created_by not in tx.active_at_start` — the creator was **not** still running when `tx` started (if it was active at our start, it committed *after* our snapshot was taken)
3. `created_by < tx.tx_id` — the creator's ID is lower than ours, meaning it started before us

Condition 3 catches transactions that both started and committed in the gap between `tx` recording its snapshot and the current moment. A transaction with a higher ID started after us, so even if it committed quickly, its writes shouldn't be in our snapshot.

**Step 4:** If none of these made `created_visible = True`, return `False`. The version was created by a still-active or future transaction.

### Phase 2: Has a visible transaction deleted this version?

If we reach this phase, the creation is visible. Now check whether a deletion has also become visible.

**Step 5 — No deletion:** If `deleted_by is None`, the version exists and is visible. Return `True`.

**Step 6 — Self-deletion:** If `deleted_by == tx.tx_id`, this transaction deleted this version. It should not see it. Return `False`.

**Step 7 — Aborted deletion:** If the deleter was aborted, the deletion never happened. The version is still visible. Return `True`.

**Step 8 — Committed deletion:** Apply the same three-way visibility test as creation: if the deleting transaction committed, was not active at our start, and has a lower ID, the deletion is in our snapshot. Return `False`.

**Step 9 — Fallback:** The deleting transaction is still in-flight or started after us. We can't see the deletion. Return `True`.

The symmetry is intentional: a creation is visible under the same rules as a deletion. The function applies the snapshot visibility rule twice — once to decide "does this version exist?" and once to decide "has it been removed?"

## Side Effects

**None.** This is a pure query function. It reads from `self._committed`, `self._aborted`, `tx.active_at_start`, and `tx.tx_id` but mutates nothing.

## Error Handling

No exceptions are raised or caught. The function assumes all inputs are valid. Specifically:
- It does not verify `tx` is active (callers like `read` check this separately via `_check_active`)
- It does not verify `version.created_by` refers to a known transaction
- If `created_by` is in neither `_committed`, `_aborted`, nor equal to `tx.tx_id`, it falls through to the "not in our snapshot" case — which is correct for still-active transactions

## Usage Patterns

Called internally by three methods:

- **`read(tx, key)`** — iterates all versions of a key, calls `_is_visible` on each, returns the last visible one's value
- **`delete(tx, key)`** — finds visible versions and marks them deleted
- **`scan(tx, prefix)`** — delegates to `read`, which calls `_is_visible`

Caller obligation: the caller must have already verified the transaction is active. `_is_visible` does not check this.

## Dependencies

- `self._committed: set` — set of committed transaction IDs
- `self._aborted: set` — set of aborted transaction IDs
- `tx.tx_id: int` — the observing transaction's ID
- `tx.active_at_start: set` — snapshot of which transactions were active when `tx` began
- `version.created_by: int` — the transaction ID that created this version
- `version.deleted_by: int | None` — the transaction ID that deleted this version (or None)

## Assumptions Not Enforced by Types

1. **Transaction IDs are monotonically increasing** — the `created_by < tx.tx_id` check relies on this to mean "started before us." If IDs were reused or non-monotonic, the visibility logic would break.
2. **`active_at_start` is an accurate snapshot** — if this set is wrong, the snapshot is wrong. Nothing enforces that `begin_transaction` populated it correctly.
3. **A transaction is in exactly one of: active, committed, aborted** — the function checks `_committed` and `_aborted` as disjoint sets. If a tx_id appeared in both, behavior would be inconsistent (the aborted check runs first, so aborted would win).
4. **`created_by` always refers to a valid transaction** — there's no check for unknown transaction IDs; they'd silently fall through as "not visible."

## Topics to Explore

- [function] `snapshot-isolation/mvcc_database.py:commit` — The write-write conflict detection that determines which transactions enter `_committed` vs `_aborted`, directly controlling what `_is_visible` reports
- [function] `snapshot-isolation/mvcc_database.py:begin_transaction` — How `active_at_start` is constructed; this snapshot is the foundation of the visibility rule
- [function] `snapshot-isolation/mvcc_database.py:garbage_collect` — Uses similar visibility reasoning to determine which versions are safe to discard
- [general] `snapshot-isolation-vs-serializable` — Snapshot isolation prevents dirty reads, non-repeatable reads, and phantom reads, but permits write skew anomalies — understanding the gap matters for knowing when SI is insufficient
- [general] `mvcc-visibility-in-postgres` — PostgreSQL's `xmin`/`xmax` system and `pg_snapshot` implement the same logical structure as this code, with `created_by` → `xmin` and `deleted_by` → `xmax`

## Beliefs

- `visibility-is-pure` — `_is_visible` is a pure function with no side effects; it reads transaction state but never modifies it
- `own-writes-always-visible` — A transaction always sees versions it created itself, regardless of commit status, ensuring read-your-own-writes consistency
- `aborted-trumps-all` — A version created by an aborted transaction is invisible to every transaction, and a deletion by an aborted transaction is ignored, effectively erasing the aborted transaction's effects
- `visibility-requires-three-conditions` — A committed transaction's writes are visible only if it is committed AND was not active at the observer's start AND has a lower transaction ID — all three must hold
- `deletion-visibility-is-symmetric` — The same three-condition rule that determines whether a creation is visible also determines whether a deletion is visible, applied independently in two phases of the same function

