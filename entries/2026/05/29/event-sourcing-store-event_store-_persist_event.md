# Function: _persist_event in event-sourcing-store/event_store.py

**Date:** 2026-05-29
**Time:** 11:15

## `_persist_event` — Append-Only Disk Serialization

### Purpose

`_persist_event` writes a single event to disk as a JSON line (newline-delimited JSON / NDJSON). It implements the durable half of the event store's append-only log — the in-memory list gives you fast reads, this method gives you crash recovery. Without it, the store is purely ephemeral.

### Contract

- **Precondition**: `self._persist_path` is a valid, writable file path (not `None`). Every caller gates on `if self._persist_path:` before calling this method — the method itself does not check.
- **Precondition**: `event` is a fully constructed `Event` with all fields populated, including a `datetime` timestamp (not a string).
- **Postcondition**: Exactly one JSON line has been appended to the file at `self._persist_path`. The file is closed (via `with`) and the write is handed off to the OS.
- **Invariant**: The on-disk log mirrors `self._events` in insertion order. Every event appended to the in-memory list gets a corresponding line on disk (assuming persistence is enabled).

### Parameters

| Parameter | Type | Notes |
|-----------|------|-------|
| `event` | `Event` | A dataclass instance. The method reads six fields from it. `metadata` may be `None`, which serializes to JSON `null`. |

No edge cases on the parameter itself — the interesting edge cases are at the I/O boundary (see Error Handling).

### Return Value

`None`. This is a fire-and-forget write. The caller (`append` / `append_batch`) does not check for success.

### Algorithm

1. Open the persist file in **append mode** (`"a"`). This creates the file if it doesn't exist and positions the write pointer at the end.
2. Build a plain `dict` from the event's fields. Notably, `timestamp` is converted from `datetime` to ISO-8601 string via `.isoformat()` — this is the only field that needs serialization beyond what `json.dumps` handles natively.
3. Serialize the dict to a JSON string and append `"\n"` to form one NDJSON line.
4. Write that line to the file. The `with` block closes the file handle immediately after.

### Side Effects

- **Disk I/O**: Opens and writes to `self._persist_path` on every call. The file is opened and closed per-event, which is safe but not high-throughput — each call incurs open/close overhead.
- **No fsync**: The write goes to the OS page cache. A crash between `write()` and the OS flushing to disk loses the event. This is a deliberate simplicity trade-off — this is a reference implementation, not a production WAL.
- **File creation**: Append mode silently creates the file on first write.

### Error Handling

None. If the file can't be opened (permissions, disk full, invalid path), the `open()` call raises `OSError`/`PermissionError` and it propagates uncaught through `append()` to the caller. The event is already in `self._events` at that point (it was appended before `_persist_event` is called), so a failure here leaves the store in an inconsistent state: in-memory has the event, disk doesn't. There is no rollback.

Similarly, `json.dumps` could theoretically fail if `event.data` contains non-serializable values (e.g., `datetime` objects nested in the data dict). This is not validated.

### Usage Patterns

Never called directly — it's a private method called from exactly two places:

1. **`append()`** — called once per single-event append, after the event is added to `self._events` and `self._streams`.
2. **`append_batch()`** — called once per event in the batch, inside the loop. Each event is persisted individually; there's no atomic batch write.

The symmetrical counterpart is `_load_from_file`, which reads the NDJSON file back during `__init__` to reconstruct the in-memory state.

### Dependencies

- **`json`** (stdlib): Serialization.
- **`datetime.isoformat()`**: Timestamp formatting. The inverse (`datetime.fromisoformat()`) is used in `_load_from_file`.
- **`self._persist_path`**: Set once in `__init__`, never modified.

### Unenforceable Assumptions

1. **`event.data` is JSON-serializable.** The type hint says `dict`, but nothing prevents values like `{key: datetime.now()}` which would blow up in `json.dumps`.
2. **`event.metadata` is JSON-serializable** (or `None`).
3. **No concurrent writers.** If two `EventStore` instances point at the same file, their appends will interleave unpredictably. There's no file locking.
4. **`_persist_path` directory exists.** Append mode creates files, not directories.

---

## Topics to Explore

- [function] `event-sourcing-store/event_store.py:_load_from_file` — The deserialization counterpart; understanding the round-trip reveals what survives a restart and what doesn't (e.g., subscriber registrations are lost)
- [function] `event-sourcing-store/event_store.py:append_batch` — Calls `_persist_event` per-event inside a loop, which means a crash mid-batch leaves a partial write on disk with no way to distinguish it from a complete batch
- [general] `ndjson-vs-sqlite-event-stores` — Why newline-delimited JSON is common for event store prototypes and where it breaks down (indexing, compaction, transactions)
- [general] `fsync-and-durability-guarantees` — The gap between `write()` returning and data actually being on disk; how production event stores (e.g., EventStoreDB) handle this
- [file] `event-sourcing-store/test_event_store.py` — How the persistence path is exercised in tests; whether crash-recovery scenarios are covered

---

## Beliefs

- `persist-event-no-fsync` — `_persist_event` writes to the OS buffer but never calls `fsync`; data can be lost on crash between write and OS flush
- `persist-event-per-call-open` — The persist file is opened and closed on every single event write, not held open across the store's lifetime
- `persist-before-notify` — Disk persistence happens before subscriber notification: an event is written to disk before any `LiveProjection` sees it
- `batch-persist-non-atomic` — `append_batch` persists events one at a time via `_persist_event`; a crash mid-batch leaves a partial batch on disk with no transactional boundary
- `persist-after-memory` — The event is appended to `self._events` before `_persist_event` is called; a write failure leaves the in-memory store ahead of disk with no rollback

