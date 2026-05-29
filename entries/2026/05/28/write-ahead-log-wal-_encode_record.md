# Function: _encode_record in write-ahead-log/wal.py

**Date:** 2026-05-28
**Time:** 18:16

# `_encode_record` — WAL Binary Record Encoder

## Purpose

`_encode_record` serializes a single WAL record into a self-describing binary format suitable for appending to a log file on disk. It exists to produce a compact, integrity-checked byte sequence that `_read_record` can later parse back — enabling crash recovery by replaying the log.

Every write operation in the WAL (put, delete, commit, checkpoint) passes through this function before hitting disk.

## Contract

**Preconditions:**
- `key` and `value` must already be encoded as `bytes` (callers handle UTF-8 encoding)
- `op_type_byte` must be a valid operation constant (1–4), though this is not enforced here
- `seq_num` must fit in an unsigned 64-bit integer (≤ 2^64 - 1)

**Postconditions:**
- Returns a byte sequence that `_read_record` can round-trip back to a `WALRecord`
- The CRC32 embedded in the output covers `op_type_byte + key + value`, so any corruption of those fields is detectable on read

**Invariant:** The first 4 bytes written to the file (by the caller, *not* this function) are the `record_length`, which equals the total size of everything *after* that 4-byte length prefix. Wait — actually, looking more carefully: the `record_length` is packed *inside* the header that this function returns, meaning the length field is part of the returned bytes, and the reader (`_read_record`) reads the length separately first, then reads `record_length` more bytes. This means the length value counts everything *after* the initial 4-byte length prefix.

## Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `seq_num` | `int` | Monotonically increasing sequence number. Packed as unsigned 64-bit (`Q`). |
| `op_type_byte` | `int` | Operation type as a raw byte value: `1`=PUT, `2`=DELETE, `3`=COMMIT, `4`=CHECKPOINT. Packed as unsigned byte (`B`). |
| `key` | `bytes` | The record key. Can be empty (e.g., for COMMIT/CHECKPOINT records). |
| `value` | `bytes` | The record value. Can be empty. |

**Edge cases:** Empty `key` and `value` are valid — `checkpoint()` and commit records pass `b""` for both.

## Return Value

A single `bytes` object containing the complete binary record. The caller writes this directly to the file descriptor. The caller is responsible for flushing/fsyncing afterward.

## Algorithm

Step by step:

1. **Compute CRC32** over `op_type_byte (1 byte) + key + value`. The `& 0xFFFFFFFF` mask ensures the result is an unsigned 32-bit value (Python's `zlib.crc32` can return signed values on some platforms).

2. **Calculate record_length** — the total byte count of everything *after* the 4-byte length prefix that `_read_record` reads first:
   - `4` (CRC) + `8` (seq_num) + `1` (op_type) + `4` (key_len) + `len(key)` + `4` (val_len) + `len(value)` = `21 + len(key) + len(value)`

3. **Pack the fixed-size header** using little-endian format `<IIQBi`:
   - `I` — `record_length` (uint32)
   - `I` — `crc` (uint32)
   - `Q` — `seq_num` (uint64)
   - `B` — `op_type_byte` (uint8)
   - `i` — `len(key)` (int32, signed)

4. **Concatenate** header + key bytes + value-length (packed as `<i`) + value bytes.

The resulting wire format is:

```
[record_length:4][crc:4][seq_num:8][op_type:1][key_len:4] | [key:N][val_len:4][value:M]
├──────────────── header (21 bytes fixed) ────────────────┘  ├──── variable ────────────┘
```

Note that `record_length` is written as part of the returned bytes but is *also* read separately by `_read_record` — the reader peeks the first 4 bytes, then reads `record_length` more bytes. So the total on-disk size per record is `4 + record_length` bytes, i.e. `25 + len(key) + len(value)`.

## Side Effects

None. This is a pure function — no I/O, no mutation, no state changes. All disk writes happen in the callers (`append`, `append_batch`, `checkpoint`, `truncate`).

## Error Handling

No explicit error handling. `struct.pack` will raise `struct.error` if values overflow their format (e.g., `seq_num` exceeds uint64 range, or key length exceeds int32 range). These would propagate to the caller.

## Usage Patterns

Called from four places:
- **`append()`** — single record writes (PUT/DELETE)
- **`append_batch()`** — multiple records + a COMMIT record, buffered into a `bytearray` before a single `write()`
- **`checkpoint()`** — writes a checkpoint marker with empty key/value
- **`truncate()`** — re-encodes kept records when rewriting WAL files

Callers always pass `key.encode("utf-8")` and `value.encode("utf-8")`, except for COMMIT and CHECKPOINT which pass `b""`.

## Dependencies

- **`struct`** (stdlib) — binary packing with format strings
- **`zlib`** (stdlib) — CRC32 checksum for integrity verification

## Notable Assumptions

1. **Key and value lengths fit in a signed 32-bit integer** (`i` format) — max ~2 GB per field. Using signed `i` rather than unsigned `I` is an odd choice; negative lengths would be nonsensical but aren't guarded against.
2. **The CRC only covers `op_type + key + value`**, not `seq_num` or `record_length`. A bit-flip in `seq_num` would go undetected. This is a deliberate trade-off (or oversight) — `_read_record` verifies CRC against the same three fields.
3. **Little-endian byte order** (`<` prefix) is hardcoded. The WAL files are not portable across architectures with different endianness, though in practice this rarely matters.
4. **`record_length` includes itself** — the value `21 + len(key) + len(value)` counts the 4 bytes for CRC but not the 4 bytes for `record_length` itself. However, `_read_record` reads the length prefix *first* (4 bytes), then reads `record_length` additional bytes — so `record_length` actually does *not* include itself, and the accounting is correct.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_record` — The inverse decoder; understanding the symmetry reveals how corruption is detected and partial writes are handled at EOF
- [function] `write-ahead-log/wal.py:WriteAheadLog.append_batch` — Shows how multiple encoded records are buffered and atomically committed with a single fsync
- [function] `write-ahead-log/wal.py:WriteAheadLog.truncate` — Re-encodes records when pruning the log, demonstrating the round-trip encode/decode contract
- [file] `write-ahead-log/test_wal.py` — Tests that exercise corruption detection, partial writes, and replay correctness
- [general] `wal-crc-coverage-gap` — The CRC excludes `seq_num` and `record_length` from integrity checking — worth understanding whether this is an intentional design choice or a gap

## Beliefs

- `wal-crc-covers-op-key-value-only` — The CRC32 checksum in each WAL record covers only `op_type_byte + key + value`; `seq_num` and `record_length` are not integrity-checked
- `wal-record-length-excludes-own-prefix` — `record_length` counts 21 + len(key) + len(value) bytes, which is everything after the 4-byte length prefix itself; total on-disk size is record_length + 4
- `wal-encode-is-pure` — `_encode_record` performs no I/O or state mutation; all disk writes are the caller's responsibility
- `wal-key-value-length-signed-int32` — Key and value lengths are packed as signed 32-bit integers (`<i`), limiting each to ~2 GB and leaving negative lengths unguarded
- `wal-little-endian-hardcoded` — The WAL binary format uses little-endian byte order unconditionally, making log files non-portable across endianness boundaries

