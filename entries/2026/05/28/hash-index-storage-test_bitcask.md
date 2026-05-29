# File: hash-index-storage/test_bitcask.py

**Date:** 2026-05-28
**Time:** 18:59

# `hash-index-storage/test_bitcask.py`

## Purpose

This is the test suite for a Bitcask storage engine implementation — a log-structured key-value store described in Chapter 3 of *Designing Data-Intensive Applications*. It validates the full lifecycle of a Bitcask store: writing, reading, updating, deleting, file rotation, compaction, hint-file generation, and crash recovery. It serves as both a correctness harness and a living specification of `BitcaskStore`'s behavioral contract.

## Key Components

### Test Functions

| Function | What it validates |
|---|---|
| `test_basic_crud` | Put, get, delete, `__len__`, `__contains__`, and missing-key returns `None` |
| `test_overwrite` | Multiple puts to the same key — only the latest value survives, length stays 1 |
| `test_file_rotation` | Writing 100 entries with `max_file_size=256` forces multiple `.data` files |
| `test_compaction` | After 100 overwrites of one key and a delete of another, `compact()` preserves only live data |
| `test_hint_files` | `compact()` produces `.hint` files, and a fresh store loads correctly from them |
| `test_startup_recovery` | Close and reopen — all data survives via normal replay |
| `test_crash_recovery` | Delete `.hint` files, then reopen — store rebuilds its in-memory index by scanning `.data` files |
| `test_large_dataset` | 10k keys, half overwritten, compacted — verifies correctness at scale |
| `test_edge_cases` | Empty store operations, delete-then-reinsert, delete of nonexistent key |

### `BitcaskStore` API (inferred contract)

```python
BitcaskStore(directory: str, max_file_size: int = ..., sync_writes: bool = True)
store.put(key: str, value: str) -> None
store.get(key: str) -> Optional[str]
store.delete(key: str) -> None
store.compact() -> None
store.keys() -> list
store.close() -> None
len(store) -> int          # via __len__
key in store -> bool       # via __contains__
```

## Patterns

**Temp directory isolation.** Every test uses `tempfile.TemporaryDirectory()` as a context manager, giving each test a clean filesystem with automatic cleanup. No test can leak state to another.

**`sync_writes=False` everywhere.** All tests disable fsync to avoid I/O bottlenecks. This is a deliberate tradeoff — tests run fast but don't exercise the durable-write path.

**Close-and-reopen idiom.** Tests like `test_startup_recovery`, `test_hint_files`, and `test_crash_recovery` create a store, close it, then open a *new* `BitcaskStore` on the same directory. This validates that the on-disk format is self-describing and the startup index-rebuild logic is correct.

**Manual test runner.** The `if __name__ == "__main__"` block runs all tests sequentially with `print("PASS: ...")` output. No pytest fixtures or decorators — tests are plain functions that assert and fail loudly.

## Dependencies

**Imports:**
- `tempfile`, `os` — standard library, for test isolation and filesystem inspection
- `bitcask.BitcaskStore` — the system under test (`hash-index-storage/bitcask.py`)

**Imported by:**
- Nothing directly. Run as a script or via pytest discovery. The sibling `tester_test_bitcask.py` likely wraps or validates these tests.

## Flow

Each test follows the same pattern:

1. Create a temp directory
2. Instantiate `BitcaskStore` pointing at it
3. Perform writes/reads/deletes
4. Assert expected state
5. (Optionally) close, mutate the filesystem, reopen, and re-assert
6. Close the store; temp directory auto-cleans

The `__main__` block runs them in a fixed order, but they have no inter-test dependencies — each is fully isolated.

## Invariants

- **Get-after-put consistency**: `put(k, v)` followed by `get(k)` must return `v`.
- **Last-writer-wins**: Multiple puts to the same key — only the final value is observable.
- **Delete is a tombstone**: After `delete(k)`, `get(k)` returns `None` and `len` decrements.
- **Compaction preserves semantics**: `compact()` must not change the observable state of any key.
- **Crash recovery completeness**: Scanning `.data` files alone (no hints) must reconstruct the same index as a clean startup with hint files.
- **File rotation is transparent**: Splitting data across multiple files has no effect on read correctness.

## Error Handling

There is none — by design. Tests assert and crash on failure. `store.get()` returns `None` for missing keys rather than raising (tested in `test_basic_crud` and `test_edge_cases`). `store.delete()` on a nonexistent key is silently accepted (tested in `test_edge_cases`). No exception-path testing exists — there are no tests for corrupted files, permission errors, or disk-full scenarios.

## Topics to Explore

- [file] `hash-index-storage/bitcask.py` — The implementation under test: on-disk format, in-memory keydir, compaction algorithm, and hint-file generation
- [function] `hash-index-storage/bitcask.py:compact` — How dead entries are garbage-collected and hint files are produced; the most complex operation in the store
- [file] `hash-index-storage/tester_test_bitcask.py` — Likely a test validator or harness wrapper; understanding its relationship to this file clarifies the testing architecture
- [file] `log-structured-hash-table/bitcask.py` — A second Bitcask implementation in the repo; comparing the two reveals which design decisions are essential vs. incidental
- [general] `bitcask-paper` — The original Bitcask design paper from Riak; understanding the keydir, merge process, and hint files gives context for what these tests are actually verifying

## Beliefs

- `bitcask-get-missing-returns-none` — `BitcaskStore.get()` returns `None` for missing or deleted keys, never raises `KeyError`
- `bitcask-delete-nonexistent-is-noop` — Calling `delete()` on a key that doesn't exist completes without error
- `bitcask-compaction-preserves-observable-state` — After `compact()`, every key returns the same value as before compaction; deleted keys remain absent
- `bitcask-crash-recovery-without-hints` — `BitcaskStore` can rebuild its in-memory index by scanning `.data` files alone when `.hint` files are missing, producing identical read results
- `bitcask-tests-disable-sync` — All tests pass `sync_writes=False`, meaning the durable fsync path is untested

