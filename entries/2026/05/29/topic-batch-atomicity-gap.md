# Topic: The NDJSON persistence has no transaction boundary for `append_batch`; worth comparing with the write-ahead-log implementation in `write-ahead-log/wal.py`

**Date:** 2026-05-29
**Time:** 11:06

## NDJSON Persistence vs. WAL: The Missing Transaction Boundary

The core issue is in `event-sourcing-store/event_store.py:69` — the `append_batch` method. It claims to "atomically append multiple events," but the persistence layer doesn't actually provide atomicity on disk.

### What the Event Store does

`append_batch` (line 69) iterates over each event and calls `_persist_event` individually (line 91):

```python
for event_type, data in events:
    # ...build event...
    if self._persist_path:
        self._persist_event(event)   # line 91: one write per event
    result.append(event)
```

`_persist_event` (line 127) simply opens the file and appends a single JSON line:

```python
def _persist_event(self, event: Event):
    with open(self._persist_path, "a") as f:
        f.write(json.dumps(record) + "\n")
```

No `flush()`. No `fsync()`. No commit marker. Each event is a separate `open`/`write`/`close` cycle. If the process crashes after writing 2 of 5 events in a batch, you get a partial batch on disk with **no way to distinguish it from a complete one** during recovery.

### What the WAL does differently

`write-ahead-log/wal.py:153` takes three precautions the event store skips:

1. **Single buffer write** — all records are serialized into one `bytearray`, then written with a single `self._fd.write(bytes(buf))` call (line 170). This minimizes the window for partial writes.

2. **Explicit COMMIT record** — after all operation records, an `OP_COMMIT` record is appended to the buffer (line 168). During recovery, any batch without a trailing COMMIT can be identified and discarded as incomplete.

3. **Forced fsync** — `self._do_sync(force=True)` at line 171 calls `flush()` + `os.fsync()`, ensuring the entire batch (including the COMMIT marker) hits stable storage before the method returns.

```python
def append_batch(self, operations):
    with self._lock:
        buf = bytearray()
        for op_type, key, value in operations:
            self._seq_num += 1
            buf.extend(_encode_record(...))
        self._seq_num += 1
        commit_seq = self._seq_num
        buf.extend(_encode_record(commit_seq, OP_COMMIT, b"", b""))  # transaction boundary
        self._fd.write(bytes(buf))       # single write
        self._do_sync(force=True)        # forced fsync
```

### Why it matters

The in-memory state in the event store is fine — `append_batch` updates `_events` and `_streams` atomically from a single-threaded perspective. The problem is the on-disk representation. When `_load_from_file` (line 138) replays the NDJSON on startup, it has no way to detect a partial batch. It will blindly load whatever lines exist, potentially leaving the store in a state that never logically existed — say, a bank transfer where the debit posted but the credit didn't.

The WAL's design makes this explicit: recovery code can scan for COMMIT records and only replay complete transactions.

## Topics to Explore

- [function] `event-sourcing-store/event_store.py:_load_from_file` — Recovery path that would silently accept partial batches; the place where a commit marker check would need to go
- [function] `write-ahead-log/wal.py:_read_record` — Binary record format with CRC checksums; shows how the WAL also detects corruption, not just incompleteness
- [general] `ndjson-fsync-gap` — None of the NDJSON writers in the codebase (partitioned-log, batch-word-count, unbundled-database) call fsync; worth auditing whether any of them need durability guarantees
- [function] `write-ahead-log/wal.py:truncate` — How the WAL garbage-collects old records after they're checkpointed; the event store has no equivalent compaction
- [general] `optimistic-concurrency-vs-durability` — The event store has `expected_version` checks (line 73) for concurrency control but no durability; these are orthogonal guarantees worth understanding separately

## Beliefs

- `event-store-append-batch-no-commit-marker` — `EventStore.append_batch` writes each event as a separate NDJSON line with no batch delimiter; a crash mid-batch produces a partial batch indistinguishable from a complete one during recovery
- `wal-append-batch-uses-commit-record` — `WriteAheadLog.append_batch` terminates every batch with an `OP_COMMIT` record, providing an explicit transaction boundary for crash recovery
- `event-store-persist-no-fsync` — `EventStore._persist_event` never calls `flush()` or `os.fsync()`; written data may remain in OS buffers at crash time
- `wal-batch-single-write-forced-sync` — `WriteAheadLog.append_batch` assembles all records into a single buffer, writes once, and forces `fsync`, minimizing the partial-write window

