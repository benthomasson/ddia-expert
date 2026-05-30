# Topic: What it would take to make snapshots survive restarts (serialize to a sidecar file, integrate with `_load_from_file`)

**Date:** 2026-05-29
**Time:** 11:16

I have enough from the observations to give a thorough answer. Here's the explanation:

---

# Making MVCC Snapshots Survive Restarts

## The Problem

`MVCCDatabase` (`snapshot-isolation/mvcc_database.py`) is entirely in-memory. Its constructor (line 49) initializes all state as plain Python dicts and sets:

```python
self._next_tx_id = 1
self._next_timestamp = 1
self._versions = {}       # key -> list[Version]
self._transactions = {}   # tx_id -> Transaction
self._committed = set()
self._aborted = set()
self._commit_timestamps = {}
```

When the process dies, everything is gone. There's no `persist_path`, no `_load_from_file`, no serialization of any kind — the grep for `serialize|to_dict|from_dict|to_json|pickle|dump|load` returned zero hits in this module.

## What Needs to Be Serialized

The challenge is that MVCC state is a graph of cross-referenced objects, not a flat log. You'd need to capture:

1. **Version chains** (`_versions`): Each `Version` (line 10–15) holds `key`, `value`, `created_by`, and `deleted_by`. These are the core data — but `created_by`/`deleted_by` are transaction IDs, so they're meaningless without the transaction metadata.

2. **Transaction metadata** (`_transactions`): Each `Transaction` (line 22–31) has `tx_id`, `start_timestamp`, `read_only`, `_status`, `write_set`, and `active_at_start`. Active transactions can't survive a restart (they'll never commit), but committed/aborted status must be preserved because `_is_visible` (line 76) checks `_committed` and `_aborted` sets to decide version visibility.

3. **Monotonic counters** (`_next_tx_id`, `_next_timestamp`): If these reset to 1 on restart, new transactions would collide with historical IDs, breaking visibility checks that compare `created_by < tx.tx_id` (line 91).

4. **Committed/aborted sets and commit timestamps**: `_committed`, `_aborted`, and `_commit_timestamps` drive every visibility decision in `_is_visible`.

## A Concrete Approach

### Step 1: Add `to_dict` / `from_dict` to `Version` and `Transaction`

```python
# Version
def to_dict(self):
    return {"key": self.key, "value": self.value,
            "created_by": self.created_by, "deleted_by": self.deleted_by}

# Transaction (only persist completed ones)
def to_dict(self):
    return {"tx_id": self._tx_id, "start_timestamp": self.start_timestamp,
            "read_only": self.read_only, "status": self._status,
            "write_set": list(self.write_set),
            "active_at_start": list(self.active_at_start)}
```

### Step 2: Add `_save_to_file` / `_load_from_file` to `MVCCDatabase`

Follow the pattern from `event-sourcing-store/event_store.py`, which accepts a `persist_path` in its constructor (line 30), calls `_load_from_file` on startup if the file exists (lines 36–37), and calls `_persist_event` on every append (lines 61–62).

For MVCC, the sidecar file would store:

```json
{
  "next_tx_id": 5,
  "next_timestamp": 5,
  "committed": [1, 3],
  "aborted": [2],
  "commit_timestamps": {"1": 2, "3": 4},
  "versions": {"x": [{"key": "x", "value": 42, "created_by": 1, "deleted_by": null}]},
  "transactions": {"1": {"tx_id": 1, "start_timestamp": 1, ...}}
}
```

### Step 3: Modify `__init__` to Accept `persist_path`

```python
def __init__(self, persist_path=None):
    self._persist_path = persist_path
    # ... initialize defaults ...
    if persist_path and os.path.exists(persist_path):
        self._load_from_file(persist_path)
```

### Step 4: Decide When to Flush

This is the hardest design decision. Options:

| Strategy | Tradeoff |
|----------|----------|
| **Flush on every commit/abort** | Safest, but slow — every `commit()` (line 181) writes the full state |
| **WAL + periodic snapshot** | Fast appends, periodic full dump, replay on restart — more complex but matches the repo's own `write-ahead-log/` module |
| **Flush only committed versions** | Skip active transactions (they can't survive restart anyway), reducing file size |

The simplest first step is flush-on-commit: in `commit()`, after updating `_committed` and `_commit_timestamps`, call `self._save_to_file()`. Similarly in `abort()`.

### Step 5: Handle Active Transactions on Recovery

Active transactions at crash time are lost. On `_load_from_file`, you'd treat them as aborted — which is safe because `_is_visible` already treats versions created by aborted transactions as invisible (line 80). You'd add their IDs to `_aborted` during recovery.

### The Key Subtlety: GC Interaction

`garbage_collect()` (referenced in test line 153) removes old versions. If you serialize *after* GC runs, the sidecar file is smaller but you've lost history. If a long-running transaction was active before the restart, its `active_at_start` set may reference versions that GC already cleaned up. Since active transactions don't survive restarts, this is actually fine — but it means **GC must run before serialization**, not after deserialization, to avoid a window where restored state includes versions that should have been collected.

## Topics to Explore

- [function] `snapshot-isolation/mvcc_database.py:garbage_collect` — Understanding what GC preserves vs. removes is critical for deciding what the sidecar file must contain
- [file] `event-sourcing-store/event_store.py` — The `_load_from_file` / `_persist_event` pattern (lines 30–62, 134+) is the closest existing persistence model in this repo to borrow from
- [file] `write-ahead-log/wal.py` — If you go the WAL route instead of full-state snapshots, this module already implements append-only log with checkpoint and replay
- [general] `mvcc-value-serialization` — The current `Version.value` can hold arbitrary Python objects; a persistence scheme forces you to constrain what types are storable (JSON-serializable, or use pickle with its security implications)
- [function] `snapshot-isolation/mvcc_database.py:_is_visible` — The visibility rules (lines 76–107) depend on `_committed`, `_aborted`, and `active_at_start` — all three must round-trip through serialization faithfully or reads return wrong results

## Beliefs

- `mvcc-no-persistence` — `MVCCDatabase` has zero serialization or persistence support; all state (`_versions`, `_transactions`, `_committed`, `_aborted`, counters) lives only in memory
- `mvcc-visibility-depends-on-tx-metadata` — `_is_visible` requires `_committed`, `_aborted`, and each transaction's `active_at_start` set; persisting version chains without this metadata makes them unreadable after restore
- `mvcc-counters-must-be-monotonic` — `_next_tx_id` and `_next_timestamp` must never reissue a previously used value, or visibility comparisons like `created_by < tx.tx_id` (line 91) silently produce wrong results
- `event-store-persist-pattern` — `EventStore` in `event-sourcing-store/event_store.py` is the repo's existing model for optional disk persistence: accept `persist_path` in constructor, load on init, append on mutation
- `mvcc-active-tx-not-recoverable` — Active (uncommitted) transactions cannot survive a restart; the safe recovery strategy is to treat them as aborted, which is consistent with `_is_visible`'s existing handling of aborted transaction versions

