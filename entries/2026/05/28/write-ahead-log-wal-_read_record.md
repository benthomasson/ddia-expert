# Function: _read_record in write-ahead-log/wal.py

**Date:** 2026-05-28
**Time:** 18:48

# `_read_record` — WAL Binary Record Deserializer

## Purpose

`_read_record` is the inverse of `_encode_record`: it reads a single binary WAL record from a file stream and returns it as a `WALRecord` dataclass. It's the fundamental deserialization primitive that every read path in the WAL depends on — recovery, replay, truncation, and iteration all call this in a loop.

It exists separately from the `WriteAheadLog` class because it's a pure function over a file handle with no dependency on WAL instance state. This makes it reusable for both normal operation and crash recovery (where the WAL object is still being constructed).

## Contract

- **Precondition**: `f` must be a binary-mode file object (`"rb"`) positioned at a record boundary. Calling this on a file positioned mid-record produces garbage.
- **Postcondition**: On success, `f` is advanced past exactly one complete record. On `None` return, `f` is at EOF or positioned past a partial record (torn write). On exception, `f` position is undefined.
- **Invariant**: A record that was written by `_encode_record` and fully persisted to disk will always round-trip through `_read_record` without CRC error.

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `f` | binary file object | An open file handle in read-binary mode, positioned at the start of a record. Not typed explicitly — the function duck-types on `.read()`. |

No validation is performed on `f`. Passing a text-mode file will raise `TypeError` from `struct.unpack`.

## Return Value

- **`WALRecord`** — on successful parse of a complete, CRC-valid record.
- **`None`** — on EOF (no bytes left) or a torn write (partial length prefix or partial record body). This is the expected signal for "no more records."
- The caller must distinguish `None` (benign end-of-data) from `ValueError` (corruption). Most callers treat `None` as "stop reading this file" and `ValueError` as "stop reading all files" (see `_read_all_records`).

## Algorithm

The on-disk format per record is:

```
[4B length][4B crc][8B seq_num][1B op_type][4B key_len][key bytes][4B val_len][value bytes]
```

Step by step:

1. **Read length prefix** (4 bytes, little-endian `uint32`). This is the byte count of everything *after* the length field itself. A short read here means EOF → return `None`.

2. **Read record body** (`record_length` bytes). A short read means a torn write (crash mid-record) → return `None`. This is the crash-tolerance mechanism: incomplete records are silently discarded.

3. **Unpack fixed header** from the record body: CRC (4B), sequence number (8B), operation type (1B), key length (4B). Format string `<IQBi` = little-endian unsigned-int, unsigned-long-long, unsigned-byte, signed-int.

4. **Extract key** — slice `key_len` bytes starting at offset 17 (4+8+1+4).

5. **Extract value length** — unpack 4 bytes as signed int at the current offset.

6. **Extract value** — slice `val_len` bytes.

7. **Verify CRC** — compute CRC32 over `op_type_byte || key || value` (same computation as `_encode_record`), mask to 32 bits. If it doesn't match the stored CRC, raise `ValueError`. This detects bit rot and partial overwrites that happened to write exactly `record_length` bytes.

8. **Construct and return** a `WALRecord` with the op type resolved from byte to string via `OP_NAMES`, and key/value decoded as UTF-8.

## Side Effects

- Advances the file position of `f` by `4 + record_length` bytes on success, or by fewer bytes on partial read.
- No mutations to any global or instance state.
- No I/O beyond the reads on `f`.

## Error Handling

| Condition | Behavior |
|-----------|----------|
| EOF before length prefix | Returns `None` |
| Torn write (partial body) | Returns `None` |
| CRC mismatch | Raises `ValueError` with sequence number |
| Invalid UTF-8 in key/value | Raises `UnicodeDecodeError` (unhandled) |
| Unknown op_type_byte | Silently maps to `"UNKNOWN"` via `OP_NAMES.get()` |

Notable: there's no protection against a `key_len` or `val_len` that exceeds `record_length`. A corrupted length field could cause a slice that reads past the buffer boundary — though in practice this just returns a short slice (Python doesn't raise on out-of-bounds slicing), and the CRC check will catch it.

## Usage Patterns

Every read-side method calls `_read_record` in the same loop pattern:

```python
while True:
    try:
        rec = _read_record(f)
        if rec is None:
            break
        # process rec
    except ValueError:
        break  # or return, stopping all reads
```

Callers include:
- **`_recover_seq_num`** — scans all files at startup to find the highest sequence number. Swallows `ValueError` per-file (skips corrupted tails).
- **`_read_all_records`** — generator that yields all valid records. On `ValueError`, it *returns* (stops reading all remaining files), treating corruption as a hard stop.
- **`truncate`** — reads all records from a file to filter by sequence number, then rewrites the file.

## Dependencies

- **`struct`** — binary packing/unpacking with format strings.
- **`zlib.crc32`** — CRC32 checksum for integrity verification.
- **`WALRecord`** — the output dataclass.
- **`OP_NAMES`** — maps op-type bytes to human-readable strings.

No external dependencies. The function is self-contained and stateless.

## Assumptions Not Enforced by Types

1. `f` is in binary mode and positioned at a record boundary — no runtime check.
2. `key_len` and `val_len` are non-negative — signed `int` is used (`<i`), so negative values would produce empty slices, pass CRC (if the original data was encoded that way), and silently succeed.
3. Key and value are valid UTF-8 — `.decode("utf-8")` will raise if they aren't, but there's no try/except around it.
4. The CRC covers `op_type + key + value` but **not** `seq_num` or `key_len`/`val_len`. A corrupted sequence number that passes CRC won't be detected. This matches `_encode_record`'s CRC computation but is a design tradeoff — sequence number corruption is silent.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_encode_record` — The encoding counterpart; understanding the exact binary layout makes the offset arithmetic in `_read_record` obvious
- [function] `write-ahead-log/wal.py:WriteAheadLog.replay` — Shows how `_read_record` results are filtered to implement committed-only replay semantics
- [function] `write-ahead-log/wal.py:WriteAheadLog.truncate` — A read-modify-write cycle that deserializes with `_read_record` and re-serializes with `_encode_record`, demonstrating the round-trip contract
- [file] `write-ahead-log/test_wal.py` — Test cases that exercise torn writes, CRC corruption, and recovery scenarios
- [general] `wal-crc-coverage-gap` — The CRC excludes `seq_num` from its computation — worth understanding whether this is intentional or a latent bug

## Beliefs

- `wal-read-record-none-on-partial` — `_read_record` returns `None` (not raises) on partial/torn writes, enabling crash tolerance by silently discarding incomplete trailing records
- `wal-crc-excludes-seq-num` — The CRC32 integrity check covers `op_type + key + value` but does not cover `seq_num`, `key_len`, or `val_len`, so corruption of those fields is undetected if the payload happens to match
- `wal-record-format-length-prefixed` — Each WAL record is prefixed with a 4-byte little-endian length that covers everything after itself, allowing the reader to skip or validate entire records atomically
- `wal-read-record-stateless` — `_read_record` is a module-level pure function with no dependency on `WriteAheadLog` instance state, enabling its use during construction (recovery) before the object is fully initialized

