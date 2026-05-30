# Topic: The two-phase write/flush model is the core architectural choice; explore how it maps to real async CDC systems (Debezium, Kafka Connect)

**Date:** 2026-05-29
**Time:** 11:25

# The Two-Phase Write/Flush Model and Real CDC Systems

## The Core Pattern in This Implementation

The CDC module in `change-data-capture/cdc.py` implements a **synchronous, embedded two-phase write** — every mutation method does both phases in a single call:

**Phase 1 — Mutate state:**
```python
# cdc.py:107-109 (insert)
t["rows"][pk] = dict(row)

# cdc.py:116-118 (update)
t["rows"][pk_value].update(changes)
```

**Phase 2 — Append to log:**
```python
# cdc.py:110 (insert)
self._log._append(table, Operation.INSERT, pk, None, dict(row))

# cdc.py:120 (update)
self._log._append(table, Operation.UPDATE, pk_value, before, after)
```

This is the architectural choice that everything downstream depends on. The write and the log entry are **atomically coupled** — there's no window where the data has changed but the log hasn't recorded it, and no window where the log has an entry that the data doesn't reflect. Both happen in the same method, in the same thread, with no I/O boundary between them.

## How This Diverges from Real CDC Systems

In production CDC (Debezium, Kafka Connect), the architecture is fundamentally different in three ways:

### 1. Log-First vs. State-First

This implementation is **state-first**: `insert()` at line 107 writes the row, then line 110 appends the log. The log is a side-effect of the write.

Debezium inverts this. It reads the database's **own** transaction log (PostgreSQL WAL, MySQL binlog) — the log that the database already writes for crash recovery. The CDC system never touches the write path. It's a passive reader of an existing artifact.

The unbundled database module (`unbundled-database/unbundled_database.py`) gets this relationship closer to right. Its `WriteAheadLog.append()` (line 47) writes the log entry first, and the `StorageEngine.apply()` (line 83) replays from the log. The log is the source of truth; the storage engine is derived.

### 2. Process Boundary and Offset Tracking

`CDCConsumer.poll()` at line 157 reads directly from an in-memory list:

```python
events = self._log.read_from(self._position)
```

The consumer and the database share a process. `self._position` (line 166) is a simple integer tracking where the consumer left off.

In Kafka Connect, this offset is the **critical durability contract**. The connector stores its offset in Kafka's internal `__consumer_offsets` topic (or in the Connect offset backing store). If the connector crashes, it resumes from the last committed offset — not from the last event it processed, but from the last one it **acknowledged**. This is why Kafka Connect has a separate "offset commit" step that this implementation doesn't need: it's the difference between `self._position = events[-1].sequence_number + 1` happening in-memory (line 165-166) versus needing to survive process death.

### 3. The Compaction Mapping

`CDCLog.compact()` at line 70 keeps only the latest event per `(table, key)` — this maps directly to **Kafka log compaction**. The semantics are identical: for any given key, you only need the most recent state. But real Kafka compaction runs as a background thread on the broker, asynchronously, and consumers must handle the fact that compaction can remove events they haven't read yet. This implementation's `compact()` has no such race because everything is synchronous and single-threaded.

## The Async Gap

The biggest thing this implementation simplifies away is the **delivery guarantee problem**. In the real two-phase model:

1. Database commits a transaction (Phase 1)
2. CDC system reads the log entry and publishes to Kafka (Phase 2)

Between steps 1 and 2, the system can crash. Debezium handles this with **exactly-once semantics** by tracking its position in the source database's log (the LSN in PostgreSQL) and committing that offset only after the event is safely in Kafka. The `CDCConsumer` here skips this entirely — position advances immediately at line 165-166 after processing, with no concept of "what if processing fails halfway through?"

The `MaterializedView.refresh()` at line 201 has the same simplification: it reads events and applies them, advancing `self._position` at the end. In a real sink connector (Kafka Connect JDBC Sink, Elasticsearch Sink), this would need to be transactional — either the derived store update AND the offset commit both succeed, or neither does.

## Topics to Explore

- [file] `unbundled-database/unbundled_database.py` — Implements WAL-first architecture where the log is the source of truth and storage is derived — closer to how real CDC systems work
- [function] `change-data-capture/cdc.py:CDCLog.compact` — Maps directly to Kafka log compaction; compare the single-threaded semantics here with Kafka's concurrent compaction thread
- [general] `offset-commit-durability` — The gap between in-memory position tracking (`consumer._position`) and durable offset storage (Kafka `__consumer_offsets`) is where most real CDC bugs live
- [function] `unbundled-database/unbundled_database.py:StorageEngine.rebuild` — The "nuclear option" for derived systems — replay the entire WAL from scratch, equivalent to resetting a Kafka Connect connector's offsets to zero
- [general] `exactly-once-delivery-in-cdc` — How Debezium achieves exactly-once by combining source LSN tracking with Kafka transactional producers — the hardest problem this implementation sidesteps

## Beliefs

- `cdc-write-and-log-are-synchronously-coupled` — Every CDCDatabase mutation (insert/update/delete) writes to in-memory state and appends to CDCLog in the same synchronous call with no failure boundary between them
- `cdc-consumer-position-is-volatile` — CDCConsumer tracks its position as an in-memory integer (`_position`) with no durable offset storage, meaning position is lost on process restart
- `cdc-log-compact-keeps-latest-per-key` — CDCLog.compact() retains only the most recent event per (table, key) pair, matching Kafka log compaction semantics but without concurrent-reader safety
- `unbundled-db-is-log-first-cdc-is-state-first` — The unbundled database writes to WAL before storage engine, while the CDC module writes to in-memory rows before appending the log — opposite ordering of the source-of-truth relationship

