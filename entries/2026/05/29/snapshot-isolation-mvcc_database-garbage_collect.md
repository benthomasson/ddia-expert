# Function: garbage_collect in snapshot-isolation/mvcc_database.py

**Date:** 2026-05-29
**Time:** 10:09

# `MVCCDatabase.garbage_collect`

## Purpose

This is the MVCC vacuum — it reclaims storage by removing version records that no active transaction can ever read. In an MVCC system, every write creates a new `Version` object rather than overwriting in place, so the version list for a key grows without bound unless something prunes it. This method is that pruning pass.

It exists because snapshot isolation requires keeping old versions alive as long as any transaction might need them for its consistent snapshot. Once every transaction that could see a version has ended, the version is dead weight.

## Contract

**Preconditions:** None enforced. Can be called at any time, even during active transactions. Does not require (or acquire) any lock — this is safe only because the implementation is single-threaded.

**Postconditions:**
- Every version created by an aborted transaction is removed.
- When no transactions are active, at most one version per key survives (the latest committed, non-deleted one).
- When transactions are active, only versions that *no* active transaction could possibly need are removed.
- The return value accurately counts how many `Version` objects were dropped.

**Invariant preserved:** After GC, `_is_visible(tx, v)` still returns the same result for every remaining version `v` and every active transaction `tx`. GC never removes a version that an active transaction could read.

## Parameters

None. It operates entirely on instance state.

## Return Value

Returns `int` — the total number of `Version` objects removed across all keys. The caller can use this to decide whether GC was productive (e.g., log it, or skip the next GC cycle if `removed == 0`).

## Algorithm

The method processes each key independently:

### Step 1: Purge aborted versions

```python
versions = [v for v in versions if v.created_by not in self._aborted]
```

Versions from aborted transactions are invisible to everyone (per `_is_visible`), so they're unconditionally dropped. This is always safe.

### Step 2a: No active transactions — aggressive compaction

When `active_txs` is empty, no snapshot references exist, so only the "current truth" matters.

It iterates the version list looking for the **last** version where:
- `created_by` is committed, AND
- either not deleted, or deleted by a transaction that hasn't committed

Because it iterates forward and overwrites `latest`, it picks the *last* such version in list order — which is the newest, since versions are appended chronologically.

If a latest surviving version is found, everything else is discarded. If *all* versions are committed-deleted, they're all removed and the key is deleted from `_versions`.

### Step 2b: Active transactions — conservative pruning

With active transactions, the method must preserve any version that *any* active transaction might read. The key insight is the `min_tx_id` — the oldest active transaction:

```python
min_tx_id = min(t.tx_id for t in active_txs)
```

If a version was committed *and* superseded (deleted or replaced) before `min_tx_id`, then *every* active transaction sees the superseding version, not this one. It's safe to remove.

Two removal conditions:

1. **Explicitly deleted:** `deleted_by` is committed and `< min_tx_id`. The deletion is visible to all active transactions, so the deleted version is invisible to all of them.

2. **Zombie (superseded):** The version is committed, not explicitly deleted, but a *newer* committed version exists (also `< min_tx_id`). The newer version shadows this one for every active transaction. The `any(...)` check scans `committed_before_oldest` to find such a superseding version.

Everything else goes into `to_keep`.

### Step 3: Update storage

If versions remain, the list is replaced. If empty, the key is deleted from `_versions` entirely to avoid accumulating empty lists.

## Side Effects

Mutates `self._versions` — removes entries and replaces version lists. Does **not** touch `_committed`, `_aborted`, `_transactions`, or any `Transaction` object. This is a pure storage compaction pass.

## Error Handling

None. No exceptions are raised or caught. The method assumes `_committed`, `_aborted`, and `_versions` are in a consistent state. If they're not (e.g., a tx_id appears in both `_committed` and `_aborted`), the behavior is silently wrong — no invariant checks.

## Usage Patterns

Typically called between transaction batches or on a periodic schedule:

```python
db = MVCCDatabase()
# ... run some transactions, commit/abort them ...
removed = db.garbage_collect()
```

**Caller obligations:** The caller should ensure no concurrent modifications to `_versions` while GC runs. In this single-threaded implementation that's trivially satisfied, but it would be a problem in a concurrent setting.

GC is idempotent — calling it twice with no intervening state changes removes 0 versions the second time.

## Dependencies

Relies entirely on instance state:
- `self._versions` — the version store being compacted
- `self._transactions` — to find active transactions
- `self._committed` / `self._aborted` — to classify transaction outcomes
- `Version.created_by`, `Version.deleted_by` — to determine version provenance

No external modules. No I/O.

## Notable Assumptions

1. **Versions are appended in tx_id order.** The no-active-transactions path picks `latest` by iterating forward, so the last match wins. If versions were out of order, this would pick the wrong "latest."

2. **`committed_before_oldest` is computed but partially unused.** The variable is built and used in the zombie check's `any(...)`, but it's not used for the explicit-deletion check. This is correct but means the variable serves a narrower purpose than its name suggests.

3. **`deleted_by` semantics are overloaded.** A version with `deleted_by` set could mean either an explicit `delete()` call or a soft-delete from a conflicting write. GC treats both the same, which is correct.

4. **Single-threaded execution.** There is no locking. If `garbage_collect` ran concurrently with `read` or `write`, you'd get races on the `_versions` dict and its lists.

---

## Topics to Explore

- [function] `snapshot-isolation/mvcc_database.py:_is_visible` — The visibility rules that GC must preserve; understanding these is prerequisite to understanding what GC can safely remove
- [function] `snapshot-isolation/mvcc_database.py:commit` — The first-committer-wins conflict detection that determines which versions end up in `_committed` vs `_aborted`
- [file] `snapshot-isolation/test_mvcc.py` — Test cases that exercise GC, especially around concurrent transactions and edge cases like all-deleted keys
- [general] `mvcc-gc-in-postgres` — How PostgreSQL's VACUUM compares: it uses xmin/xmax horizons and a visibility map, solving the same problem at scale with concurrent access
- [general] `write-skew-and-gc-interaction` — Whether GC can interfere with serializable snapshot isolation (SSI) conflict detection if it removes versions needed for read-set validation

## Beliefs

- `gc-preserves-active-snapshots` — `garbage_collect` never removes a version that `_is_visible` would return `True` for against any currently active transaction
- `gc-unconditionally-purges-aborted` — All versions with `created_by` in `_aborted` are removed regardless of active transaction state
- `gc-no-active-keeps-one` — When no transactions are active, GC retains at most one version per key (the latest committed non-deleted version)
- `gc-uses-min-txid-horizon` — With active transactions, the oldest active `tx_id` defines the GC horizon; only versions superseded before that horizon are eligible for removal
- `gc-not-thread-safe` — `garbage_collect` mutates `_versions` without synchronization, assuming single-threaded execution

