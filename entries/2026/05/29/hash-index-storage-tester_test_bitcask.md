# File: hash-index-storage/tester_test_bitcask.py

**Date:** 2026-05-29
**Time:** 06:56

# `hash-index-storage/tester_test_bitcask.py`

## Purpose

This is the **independent verification test suite** for the Bitcask hash-index storage engine. The `tester_` prefix distinguishes it from the implementation's own `test_bitcask.py` — this file was written by a separate "tester" agent to validate the implementation against its spec, providing a second pair of eyes. It exercises `BitcaskStore` end-to-end: CRUD, file rotation, compaction, hint-file recovery, crash recovery, scale, and edge cases.

The file is designed to run both as a pytest module and standalone via `__main__`, printing `PASS: <name>` per test for quick visual confirmation.

## Key Components

### Test Functions

| Function | What it validates |
|---|---|
| `test_basic_crud` | Full lifecycle: `put`, `get`, `delete`, `keys`, `len`, `__contains__` on a fresh store |
| `test_overwrite` | Multiple writes to the same key return the latest value; `len` stays 1 |
| `test_file_rotation` | When `max_file_size=256` is exceeded, multiple `.data` files appear on disk; all keys remain readable |
| `test_compaction` | After 100 overwrites + a delete, `compact()` preserves only live data and removes tombstones |
| `test_hint_files_and_startup` | `compact()` produces `.hint` files; a new `BitcaskStore` instance on the same directory rebuilds correctly from hints |
| `test_startup_recovery_no_hints` | Manually deletes `.hint` files, then reopens — forces full data-file scan recovery and verifies correctness |
| `test_large_dataset` | 10k keys with 5k overwrites; spot-checks both updated and untouched keys |
| `test_edge_cases` | Empty-store operations, delete of nonexistent key (no crash), put-delete-put cycle |
| `test_example_from_spec` | Runs the exact scenario from the task specification as a golden-path regression |

### Shared Setup Pattern

Every test follows the same structure:
1. Create a `tempfile.TemporaryDirectory`
2. Instantiate `BitcaskStore(dir, sync_writes=False, ...)`
3. Exercise operations and assert
4. Explicitly call `s.close()`
5. Print `PASS: <name>`

`sync_writes=False` is used everywhere to avoid fsync overhead in tests — the tests care about logical correctness, not durability guarantees.

## Patterns

- **Temp-dir isolation**: Each test gets its own directory via `tempfile.TemporaryDirectory`, so tests are fully independent and leave no filesystem residue. The `with` block handles cleanup.
- **Dual runner**: Tests are valid pytest functions (discovered by name) *and* runnable standalone via the `__main__` block. The `print("PASS: ...")` lines provide human-readable output when running outside pytest.
- **Reopen-and-verify**: `test_hint_files_and_startup` and `test_startup_recovery_no_hints` both close the store and open a new instance on the same directory — this is the critical pattern for testing persistence and recovery, since Bitcask rebuilds its in-memory index on startup.
- **Controlled file size**: Several tests set `max_file_size` to small values (256, 512, 1024) to force file rotation without writing megabytes of data.

## Dependencies

**Imports:**
- `os` — listing directory contents to verify `.data` and `.hint` file creation, removing hint files
- `tempfile` — isolated test directories
- `bitcask.BitcaskStore` — the system under test

**Imported by:** Nothing imports this file. It's a leaf test module.

## Flow

When run standalone (`python tester_test_bitcask.py`), tests execute sequentially in declaration order. Each test is self-contained: create dir → open store → operate → assert → close → print. No shared state between tests.

The two recovery tests (`test_hint_files_and_startup`, `test_startup_recovery_no_hints`) have a two-phase flow: write data with one instance, close it, then open a second instance and verify the data survived the restart. This tests the startup index-rebuild path.

## Invariants

- **Latest-write-wins**: `test_overwrite` and `test_compaction` verify that repeated `put` calls to the same key always yield the most recent value.
- **Delete is permanent until re-put**: After `delete("a")`, `get("a")` returns `None`, `"a" not in s`, and `len` decrements. `test_edge_cases` verifies a subsequent `put` resurrects the key.
- **Compaction preserves only live data**: After `compact()`, deleted keys stay deleted, and the latest value for live keys is unchanged.
- **Recovery is lossless**: Both hint-based and scan-based recovery must reconstruct the exact same logical state as the pre-close store.
- **File rotation is transparent**: Readers see all keys regardless of how many `.data` files exist.

## Error Handling

There is no explicit error handling — tests rely on assertions and will raise `AssertionError` on failure. The design assumes `BitcaskStore` doesn't raise on `delete` of a nonexistent key (verified in `test_edge_cases`). The `with tempfile.TemporaryDirectory()` blocks ensure cleanup even if a test fails mid-execution.

## Topics to Explore

- [file] `hash-index-storage/bitcask.py` — The implementation under test; understanding `put`/`get`/`compact`/hint-file format is essential context
- [file] `hash-index-storage/test_bitcask.py` — The implementation's own test suite; compare coverage and approach with this tester file
- [general] `bitcask-hint-file-format` — How hint files encode the keydir for fast startup without scanning all data files
- [function] `hash-index-storage/bitcask.py:compact` — The compaction algorithm that merges data files, removes dead entries, and emits hint files
- [file] `log-structured-hash-table/bitcask.py` — A second Bitcask implementation in the repo; compare design choices between the two

## Beliefs

- `tester-bitcask-sync-writes-disabled` — All tester tests pass `sync_writes=False` to BitcaskStore, testing logical correctness without fsync overhead
- `tester-bitcask-recovery-two-paths` — The test suite verifies two distinct recovery paths: hint-file-based rebuild and full data-file scan (with hints deleted)
- `tester-bitcask-compaction-removes-tombstones` — After `compact()`, deleted keys remain inaccessible and `len` reflects only live keys
- `tester-bitcask-file-rotation-transparent` — File rotation (triggered by `max_file_size`) is invisible to readers — all keys remain retrievable across multiple `.data` files
- `tester-bitcask-delete-nonexistent-safe` — Calling `delete()` on a key that doesn't exist must not raise an exception

