# Topic: DDIA's discussion of MVCC and snapshot isolation explains the principled solution: readers operate on an immutable view while writers create new versions

**Date:** 2026-05-29
**Time:** 07:31

# MVCC and Snapshot Isolation: Readers See an Immutable Past, Writers Create New Versions

DDIA's core insight about snapshot isolation is that you can avoid read-write conflicts entirely by giving each transaction a frozen view of the database at the moment it started. Writers never modify existing data — they append new versions. The `MVCCDatabase` in this codebase implements exactly that pattern.

## The Version Chain

Every value in the database exists as a `Version` object (`snapshot-isolation/mvcc_database.py:12-17`). A version records *who created it* and *who deleted it*:

```python
class Version:
    def __init__(self, key, value, created_by, deleted_by=None):
```

The database stores these as append-only lists per key (`_versions`: a `dict` mapping keys to `list[Version]`, line 55). When a transaction writes a key, it doesn't touch the old version — it appends a new `Version` to the list (lines 143-149). When it deletes, it doesn't remove anything — it stamps `deleted_by` on the existing visible version (lines 157-162). Old versions remain intact for concurrent readers.

This is DDIA's "writers create new versions" principle made concrete: the `_versions` lists grow monotonically, and the only mutations are setting `deleted_by` on a version or updating a version the *same* transaction already created (line 140-141, a self-write optimization).

## The Snapshot: What Each Transaction Can See

The snapshot is defined by the visibility function `_is_visible` (lines 77-112). This is the heart of MVCC. A version is visible to transaction `tx` if and only if:

1. **The creating transaction committed before `tx` started** — checked via `created_by in self._committed`, `created_by not in tx.active_at_start`, and `created_by < tx.tx_id` (lines 91-95). This three-part check implements DDIA's snapshot rule: you see committed data from transactions that finished before your snapshot, but not from transactions that were still in-flight.

2. **Own writes are always visible** — `created_by == tx.tx_id` (line 89). A transaction must see its own uncommitted changes.

3. **Aborted transactions are invisible** — `created_by in self._aborted` short-circuits to `False` (lines 83-84).

4. **Deletion visibility mirrors creation visibility** — lines 98-112 apply the same committed/active/ordering checks to `deleted_by`. If the deleting transaction isn't yet visible, the version still appears alive.

The `active_at_start` set (captured at `begin_transaction`, lines 72-74) is DDIA's snapshot descriptor — it records which transactions were concurrent, so the visibility check can exclude their writes even after they commit.

## Read Path: A Frozen View

`read()` (lines 118-127) iterates through all versions of a key and returns the last visible one. Because visibility is determined entirely by the transaction's `start_timestamp` and `active_at_start` snapshot, the result never changes regardless of what other transactions do concurrently. The test at line 22 (`test_example_usage`) demonstrates this directly: `tx2` reads `account_A` as 1000, then `tx3` commits a write changing it to 900, and `tx2` *still* reads 1000.

## Write Conflicts: First-Committer-Wins

Snapshot isolation doesn't prevent all anomalies — it specifically targets read consistency while detecting write-write conflicts at commit time. The `commit()` method (lines 168-187) implements first-committer-wins: if another transaction wrote to the same key and committed during our lifetime (was in `active_at_start` or started after us), we abort. This is tested in `test_4_write_write_conflict` (test file lines 68-79).

## SSI: Extending the Model for Serializability

The `ssi_database.py` builds on the same MVCC foundation but adds read-write conflict detection for write skew — the anomaly snapshot isolation alone cannot prevent. It uses `_visible_value` (line 59) with timestamp-based version lookup instead of the `active_at_start` set approach, and adds predicate locks (`_predicate_locks`, line 14) and dependency tracking (`_dependency_graph`, line 47) to detect when concurrent transactions read-then-write in ways that violate serializability.

## Garbage Collection: The Practical Cost of Immutability

DDIA notes that keeping old versions forever is impractical. The `garbage_collect` method (referenced in `test_9_gc_removes_unreachable`, test file line 147) reclaims versions that no active transaction can see — but `test_10_gc_preserves_active` (line 161) confirms it preserves versions still needed by long-running snapshots. This is the operational tension DDIA highlights: immutable snapshots are elegant but require background cleanup.

---

## Topics to Explore

- [function] `snapshot-isolation/mvcc_database.py:commit` — The write-write conflict detection logic and how first-committer-wins interacts with the `active_at_start` snapshot
- [file] `write-skew-detection/ssi_database.py` — How SSI extends snapshot isolation with read-write conflict detection and predicate locks to achieve serializability
- [function] `snapshot-isolation/mvcc_database.py:garbage_collect` — How the system reclaims old versions without breaking active snapshots (read the remaining lines 200-317)
- [general] `snapshot-vs-ssi-tradeoffs` — Compare the two implementations: `MVCCDatabase` uses `active_at_start` sets while `SSIDatabase` uses timestamp ordering — why the different approaches?
- [function] `write-skew-detection/ssi_database.py:read_predicate` — How predicate locks detect phantom reads, the class of anomaly that point-level conflict checks miss

## Beliefs

- `mvcc-append-only-versions` — Writers never modify existing Version objects (except self-overwrites); new values are appended to per-key version lists, preserving the immutable snapshot for concurrent readers
- `visibility-requires-three-conditions` — A version is visible only if its creator committed, was not in the reader's `active_at_start` set, and has a lower `tx_id` — all three must hold
- `first-committer-wins-on-write-conflict` — When two concurrent transactions write the same key, the first to call `commit()` succeeds and the second is aborted automatically
- `gc-preserves-active-snapshots` — Garbage collection removes only versions unreachable by any active transaction, ensuring long-running read-only transactions see consistent data
- `ssi-extends-mvcc-with-dependency-tracking` — `SSIDatabase` adds a `_dependency_graph` and `_predicate_locks` on top of MVCC to detect read-write conflicts (write skew) that basic snapshot isolation permits

