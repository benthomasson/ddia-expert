# File: write-skew-detection/ssi_database.py

**Date:** 2026-05-29
**Time:** 10:08

## Purpose

This file implements **Serializable Snapshot Isolation (SSI)** — the concurrency control algorithm described in DDIA Chapter 7 that upgrades snapshot isolation to full serializability by detecting write skew at commit time. It owns the entire SSI lifecycle: MVCC versioned storage, transaction read/write tracking, and the three-pronged conflict detection (write-write, read-write, and phantom) that makes SSI work.

The key insight SSI addresses: plain snapshot isolation prevents dirty reads and lost updates but allows **write skew**, where two transactions read overlapping data, make decisions based on it, then write to *different* keys — each individually valid, but together violating an invariant. The classic example is two on-call doctors each checking "are there at least two doctors on call?" then removing themselves, leaving zero.

## Key Components

### `SSITransaction`

A transaction's tracking state. Holds:

- **`tx_id` / `start_timestamp`** — identity and the MVCC snapshot point. The start timestamp determines what versions are visible via `_visible_value`.
- **`_read_set`** — keys this transaction has read. Used during commit to detect read-write conflicts (another transaction wrote a key we read).
- **`_write_set`** — keys this transaction intends to write. Used for write-write conflict detection.
- **`_writes` / `_deletes`** — buffered mutations, not applied to the store until commit succeeds. This is the "optimistic" part: we do work locally and validate at the end.
- **`_predicate_locks`** — list of `(predicate_fn, snapshot_result_dict)` tuples. These capture *range reads* — not just which keys were read, but the predicate that selected them, so the database can re-evaluate it at commit time to detect phantoms (new rows that would have matched).
- **`_status`** — state machine: `"active"` → `"committed"` or `"aborted"`.

The properties (`read_set`, `write_set`, `predicate_locks`) return defensive copies, preventing external mutation of internal state.

### `SSIDatabase`

The database engine. Core state:

- **`_store`** — MVCC versioned storage: `key → [(commit_ts, value, tx_id), ...]`. Each key accumulates versions; reads pick the latest version visible at a given timestamp. Deletions are stored as a `_DELETED` sentinel rather than removing the key, preserving the version history.
- **`_committed_txns`** — append-only log of committed transaction objects. This is the "recent enough" set that commit validation scans for concurrent transactions.
- **`_constraints`** — named invariant functions checked at commit time (e.g., "at least one doctor on call").
- **`_dependency_graph`** — tracks causal `tx_id → {tx_ids it read from}`, useful for external analysis of serialization order.
- **`_pessimistic`** — mode flag. When true, read-write conflicts are detected eagerly (at read time) rather than at commit.

### `_DeletedSentinel` / `_DELETED`

Module-level singleton used as a tombstone value in the MVCC store. Compared by identity (`is _DELETED`), not equality.

## Patterns

**Optimistic Concurrency Control (OCC):** Transactions buffer all writes locally in `_writes`/`_deletes`. No locks are taken during execution. Conflict detection happens entirely in `commit()`, and the transaction is either fully installed or fully aborted — no partial state.

**MVCC (Multi-Version Concurrency Control):** Each key stores a list of `(timestamp, value, tx_id)` tuples. `_visible_value` scans this list to find the latest version at or before a snapshot timestamp. This gives each transaction a consistent point-in-time view without blocking writers.

**Predicate Locks for Phantom Detection:** Rather than locking individual rows, `read_predicate` stores the predicate function itself along with its result set. At commit time, the predicate is re-evaluated against current state. If the matching keys or values differ, a phantom conflict is raised. This is the mechanism that catches write skew scenarios involving range queries.

**Dual Mode (Optimistic/Pessimistic):** The `_pessimistic` flag switches between SSI's standard optimistic approach (detect at commit) and eager detection (abort at read time via `_check_read_conflict`). This lets the same codebase demonstrate both strategies.

## Dependencies

**Imports:** None — this is a zero-dependency, self-contained module.

**Imported by:**
- `test_ssi.py` — test suite exercising SSI behavior
- `tester_test_ssi.py` — meta-test validating the test suite

## Flow

### Transaction lifecycle

1. **`begin_transaction()`** — allocates a unique `tx_id` and `start_timestamp`, returns an `SSITransaction`.

2. **Reads** — `read(tx, key)` checks buffered writes/deletes first (read-your-own-writes), then falls back to `_visible_value` for the MVCC snapshot. Tracks the key in `_read_set` and records a causal dependency on the writing transaction. In pessimistic mode, immediately checks for concurrent writes to this key.

3. **Predicate reads** — `read_predicate(tx, predicate)` builds a full snapshot, merges in buffered writes, evaluates the predicate over all key-value pairs, stores the predicate + results for later phantom detection.

4. **Writes** — `write(tx, key, value)` and `delete(tx, key)` buffer changes locally. `_writes` and `_deletes` are kept mutually exclusive per key.

5. **`commit(tx)`** — the critical path. Runs four checks in order:
   - **Write-write conflicts:** Do any concurrent committed transactions write the same keys we do?
   - **Read-write conflicts (write skew):** Did any concurrent committed transaction write a key that we *read*? This is the SSI-specific check that plain SI lacks.
   - **Phantom detection:** Re-evaluate all stored predicates. If the matching key set or values changed, a phantom is detected.
   - **Constraint validation:** If no conflicts so far, apply writes to a hypothetical snapshot and check all registered constraints.

   If any conflicts are found, the transaction is aborted with a structured conflict list. Otherwise, writes are installed in `_store` with a new commit timestamp.

### Conflict priority in `_conflict_reason`

When multiple conflict types exist simultaneously, the reason message follows a priority: write-write > phantom > read-write > constraint. This reflects severity — a write-write conflict is the most direct, while a constraint violation is a semantic check layered on top.

## Invariants

- **Snapshot consistency:** A transaction only sees versions with `commit_ts <= start_timestamp`. This is enforced in `_visible_value` and `_snapshot`.
- **Read-your-own-writes:** Buffered writes/deletes are checked before the MVCC store in `read()`, ensuring a transaction always sees its own modifications.
- **Mutual exclusion of writes/deletes per key:** `write()` discards any pending delete for the same key, and `delete()` discards any pending write. A key is in `_writes` XOR `_deletes`, never both.
- **Status gating:** All read/write operations check `tx._status == "active"` and raise `RuntimeError` if the transaction is already committed or aborted.
- **Read-only optimization:** Transactions with an empty write set skip all conflict detection in `commit()` — they can't cause write skew because they don't write.
- **Committed transactions are immutable:** Once a transaction is appended to `_committed_txns`, its state is never modified.

## Error Handling

- **Active-status guard:** `read`, `read_predicate`, `write`, `delete` all raise `RuntimeError` if the transaction isn't active. `commit` returns a failure dict instead of raising.
- **Predicate evaluation:** `read_predicate` wraps each `predicate(k, v)` call in a try/except, silently skipping keys where the predicate throws. Same during phantom re-evaluation in `commit`. This is pragmatic — predicates are user-supplied functions that might not handle all value types.
- **Pessimistic aborts:** `_check_read_conflict` raises `RuntimeError` immediately, short-circuiting the transaction. This is by design — in pessimistic mode, the caller needs to catch and retry.
- **Commit result:** Returns a structured dict `{"committed": bool, "reason": str|None, "conflicts": list}` rather than raising. This lets callers inspect *what* conflicted without exception handling, which is important for retry logic.

## Topics to Explore

- [file] `write-skew-detection/test_ssi.py` — See the concrete write skew scenarios (doctors on-call, meeting room double-booking) that exercise each conflict type
- [file] `snapshot-isolation/mvcc_database.py` — The plain MVCC implementation without SSI; comparing the two shows exactly what SSI adds
- [general] `predicate-lock-vs-index-range-lock` — How real databases (PostgreSQL's SSI) approximate predicate locks with index-range locks for performance, and why this implementation uses exact predicates instead
- [function] `write-skew-detection/ssi_database.py:commit` — Walk through the four-phase validation with a specific write skew scenario to see how read-write conflict detection catches it
- [general] `ssi-false-positive-rate` — SSI can abort transactions that would have been serializable; explore when this implementation over-aborts and how production systems minimize false positives

## Beliefs

- `ssi-read-only-skip-validation` — Read-only transactions (empty write set) bypass all conflict detection and always commit successfully
- `ssi-buffered-writes-not-visible-to-others` — A transaction's writes are buffered in `_writes`/`_deletes` and only installed in `_store` after successful commit validation; other concurrent transactions never see uncommitted writes
- `ssi-phantom-detection-uses-predicate-reevaluation` — Phantom detection works by re-evaluating the original predicate function against current committed state, comparing key sets and values to the snapshot-time result
- `ssi-conflict-detection-scans-all-committed` — The commit method scans all entries in `_committed_txns` with `commit_timestamp > tx.start_timestamp` to find concurrent transactions; there is no index or pruning of old committed transactions
- `ssi-pessimistic-mode-aborts-on-read` — When `pessimistic=True`, `_check_read_conflict` raises `RuntimeError` during `read()`/`read_predicate()` if a concurrent committed transaction modified the key, rather than deferring detection to commit time

