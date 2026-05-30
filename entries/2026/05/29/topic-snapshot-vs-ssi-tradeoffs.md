# Topic: Compare the two implementations: `MVCCDatabase` uses `active_at_start` sets while `SSIDatabase` uses timestamp ordering — why the different approaches?

**Date:** 2026-05-29
**Time:** 10:10

## Why `active_at_start` Sets vs. Timestamp Ordering

The difference comes down to **when writes become visible in the shared store**, which in turn dictates what metadata each transaction needs to reconstruct its snapshot.

### MVCCDatabase: Eager Writes Require Active-Set Tracking

`MVCCDatabase` writes versions **immediately** into the shared `_versions` dict, tagged only with the creating transaction's ID (`mvcc_database.py:148–155`). At the moment of the write, the transaction hasn't committed yet — so the store contains a mix of committed, uncommitted, and aborted versions all jumbled together.

This creates an ambiguity that timestamps alone can't resolve. Consider: transaction 3 starts, then transaction 5 starts, then transaction 3 commits. When transaction 5 reads, it needs to see transaction 3's writes — but transaction 3 was still active when transaction 5 began. Without `active_at_start`, transaction 5 has no way to distinguish "committed before my snapshot" from "started before me but committed after me."

The visibility rule at `mvcc_database.py:82–89` shows all three conditions:

```python
if created_by in self._committed:
    if created_by not in tx.active_at_start:   # wasn't in-flight at my start
        if created_by < tx.tx_id:              # started before me
            created_visible = True
```

The `active_at_start` set is the mechanism that captures the state of the world at snapshot time. It's populated eagerly at `begin_transaction` (`mvcc_database.py:66–68`) and then never changes — it's a frozen snapshot of which transactions were in-flight.

### SSIDatabase: Deferred Writes Make Timestamps Sufficient

`SSIDatabase` takes the opposite approach: writes are **buffered** in `tx._writes` (`ssi_database.py:152–155`) and only materialized into `_store` at commit time, stamped with the `commit_timestamp` (`ssi_database.py` commit path). The store therefore contains **only committed data**, and each version carries the exact timestamp at which it became durable.

This makes visibility trivial — `_visible_value` at `ssi_database.py:63–73` just finds the latest version where `commit_ts <= snapshot_ts`:

```python
for commit_ts, value, tx_id in versions:
    if commit_ts <= snapshot_ts and (best is None or commit_ts > best[0]):
        best = (commit_ts, value, tx_id)
```

No active-set tracking needed. If a version is in the store, it's committed. If its commit timestamp is before your start timestamp, it's visible. The total ordering of timestamps resolves all ambiguity that the MVCC implementation needs the active set for.

### Why Each Approach Fits Its Module

The choice isn't arbitrary — it aligns with what each module is demonstrating:

- **MVCCDatabase** is modeling the PostgreSQL-style MVCC approach described in DDIA, where the database maintains version chains and transactions filter them at read time. The `active_at_start` set is the canonical mechanism from that design (PostgreSQL's `xmin`/`xmax` with the `pg_snapshot` active-transaction list).

- **SSIDatabase** builds on snapshot isolation but adds **write skew detection**, which requires comparing transaction timelines. Timestamp ordering makes the "concurrent transactions" query natural — `commit_timestamp > tx.start_timestamp` at `ssi_database.py:~183` directly identifies the danger window. The deferred-write model also means SSI conflict checks run against clean committed state, not against a mix of tentative and finalized versions.

The tradeoff: `active_at_start` sets grow with concurrency (one entry per in-flight transaction) but give precise per-version visibility. Timestamp ordering is O(1) for visibility but requires all writes to be deferred until commit, which means reads can't see own-writes without consulting the buffer first (`ssi_database.py:93–97`).

## Topics to Explore

- [function] `snapshot-isolation/mvcc_database.py:commit` — How first-committer-wins uses `active_at_start` for write-write conflict detection at commit time
- [function] `write-skew-detection/ssi_database.py:read_predicate` — How predicate locks enable phantom detection, which timestamp ordering alone couldn't catch
- [function] `snapshot-isolation/mvcc_database.py:garbage_collect` — How GC must respect active transactions' snapshots — the `active_at_start` approach makes this harder since old versions may still be needed
- [general] `deferred-vs-eager-write-buffering` — The read-own-writes problem: MVCCDatabase gets it for free (versions are in the store), SSIDatabase must check `tx._writes` on every read
- [function] `write-skew-detection/ssi_database.py:_check_read_conflict` — The pessimistic mode path — how SSI can optionally detect conflicts at read time instead of waiting for commit

## Beliefs

- `mvcc-active-set-frozen-at-begin` — `MVCCDatabase.begin_transaction` captures `active_at_start` as a frozen set of all currently active tx IDs; it is never modified after creation
- `ssi-store-contains-only-committed-data` — `SSIDatabase._store` only contains versions with commit timestamps; uncommitted writes live in `tx._writes` until commit
- `mvcc-visibility-requires-three-conditions` — A version is visible in MVCCDatabase only if its creator is committed, was NOT in `active_at_start`, and has a tx_id less than the reader's
- `ssi-visibility-is-timestamp-comparison` — `SSIDatabase._visible_value` determines visibility by a single comparison: `commit_ts <= snapshot_ts`, with no active-set tracking
- `ssi-own-writes-require-buffer-check` — `SSIDatabase.read` must check `tx._writes` and `tx._deletes` before consulting the store, since own writes aren't materialized until commit

