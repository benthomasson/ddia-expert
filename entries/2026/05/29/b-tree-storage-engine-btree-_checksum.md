# Function: _checksum in b-tree-storage-engine/btree.py

**Date:** 2026-05-29
**Time:** 12:07



## `WAL._checksum` — CRC32 Integrity Check

### Purpose

`_checksum` computes a 32-bit CRC checksum over a byte buffer. It exists to detect corruption in WAL entries — each entry is written with a checksum appended, and during recovery the checksum is recomputed and compared to verify the data wasn't partially written or corrupted on disk.

### Contract

- **Precondition**: `data` must be a bytes-like object (anything `zlib.crc32` accepts).
- **Postcondition**: Returns a deterministic, unsigned 32-bit integer in the range `[0, 0xFFFFFFFF]`.
- **Invariant**: For identical input bytes, the output is always identical — pure function, no state.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `data` | `bytes` | The raw page data to checksum. In practice this is always a full page-sized buffer (padded to `page_size` bytes). |

The method is `@staticmethod` — it takes no `self` and captures no instance state.

### Return Value

An `int` in `[0, 2^32 - 1]`. The `& 0xFFFFFFFF` mask exists because Python 2's `zlib.crc32` could return signed values (negative for inputs where bit 31 is set). In Python 3 this is technically unnecessary — `crc32` already returns an unsigned int — but the mask is a defensive idiom that ensures portability.

### Algorithm

1. Pass `data` through `zlib.crc32`, which computes the CRC-32 checksum per the ISO 3309 / ITU-T V.42 polynomial.
2. Bitwise AND with `0xFFFFFFFF` to clamp to an unsigned 32-bit range.

That's it — one line of computation.

### Side Effects

None. Pure function, no I/O, no mutation.

### Error Handling

If `data` is not a bytes-like object, `zlib.crc32` raises `TypeError`. This is not caught — it propagates to the caller. No other failure modes exist.

### Usage Patterns

Used in exactly two places within `WAL`:

1. **`log_write`** (line ~114): computes the checksum and appends it as 4 bytes (`struct.pack('>I', ...)`) after the page data in the WAL entry.
2. **`recover`** (line ~135): recomputes the checksum over the recovered page data and compares it to the stored checksum. If they don't match, the entry is silently skipped — the WAL treats a checksum mismatch as a torn write and stops replaying at that point.

This means the checksum serves as the sole mechanism for distinguishing complete WAL entries from partially-written ones after a crash.

### Dependencies

- `zlib` (stdlib) — for `crc32`.

### Assumptions

- CRC-32 is sufficient for corruption detection. It is not cryptographic — it won't detect intentional tampering, but that's not the threat model. The threat model is torn writes and bit-rot during crash recovery.
- The checksum covers only the page data, not the WAL entry header (seq, page_num, data_len). A corrupted header could cause the recovery loop to read the wrong `data_len`, but this would almost certainly cause either a bounds error or a checksum mismatch, so it's implicitly safe.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:WAL.recover` — See how checksum mismatches terminate the recovery loop and what that means for partial writes
- [function] `b-tree-storage-engine/btree.py:WAL.log_write` — The write side: how entries are structured and fsynced before being considered durable
- [general] `crc32-vs-xxhash-for-wal` — CRC-32 is cheap but has known collision weaknesses at scale; understand when storage engines upgrade to stronger checksums
- [function] `b-tree-storage-engine/btree.py:BTree._wal_write_page` — The caller that coordinates WAL logging with page writes, showing the full write path

## Beliefs

- `wal-checksum-is-crc32` — WAL integrity checking uses CRC-32 via `zlib.crc32`; no cryptographic hash is involved
- `wal-checksum-covers-data-only` — The checksum covers only the page data bytes, not the WAL entry header (sequence number, page number, data length)
- `wal-recovery-skips-bad-checksum` — During recovery, a WAL entry whose recomputed checksum doesn't match the stored checksum is silently discarded and recovery stops
- `checksum-mask-is-python2-compat` — The `& 0xFFFFFFFF` mask is a Python 2 portability idiom; Python 3's `zlib.crc32` already returns unsigned values

