# Topic: What the WAL on-disk format looks like, and how replay handles partial writes or corruption

**Date:** 2026-05-28
**Time:** 19:06

I have enough from the observations to give a thorough explanation. Let me trace through the code.

---

# WAL On-Disk Format and Corruption-Tolerant Replay

## On-Disk Format

Each WAL file (`*.wal` in the log directory) is a sequence of back-to-back binary records with no file-level header. The record format is defined in `_encode_record` (`wal.py:28–34`):

```
┌──────────────┬──────────┬──────────┬──────────┬──────────┬───────┬──────────┬───────┐
│ record_length │   CRC32  │  seq_num │ op_type  │ key_len  │  key  │ val_len  │ value │
│   4 bytes     │  4 bytes │  8 bytes │  1 byte  │  4 bytes │  var  │  4 bytes │  var  │
│   uint32 LE   │ uint32 LE│ uint64 LE│  uint8   │ int32 LE │ bytes │ int32 LE │ bytes │
└──────────────┴──────────┴──────────┴──────────┴──────────┴───────┴──────────┴───────┘
```

**`record_length`** is the total byte count of everything *after* this field — from CRC through the end of value. This is computed at `wal.py:31`:

```python
record_length = 4 + 8 + 1 + 4 + len(key) + 4 + len(value)
```

That's `CRC(4) + seq_num(8) + op_type(1) + key_len(4) + key(var) + val_len(4) + value(var)`.

**`CRC32`** covers `op_type + key + value` only (`wal.py:30`). Notably, it does **not** cover `seq_num` or the length fields — it verifies payload integrity, not framing integrity.

**`op_type`** is one of four byte values (`wal.py:10–13`): `PUT(1)`, `DELETE(2)`, `COMMIT(3)`, `CHECKPOINT(4)`.

### Batches and Commits

Batched writes (`append_batch`, `wal.py:148–162`) serialize multiple records into a single buffer, appending a `COMMIT` record at the end. The entire buffer is written in one `write()` call and then force-synced. This means a batch is only considered complete if the `COMMIT` record is present and intact.

### File Rotation

WAL files are named `000001.wal`, `000002.wal`, etc. (`wal.py:110`). When a file exceeds `max_file_size` (default 10 MB), `_maybe_rotate` triggers creation of the next numbered file (`wal.py:103–110`). The old file is flushed and fsynced before closing.

## How Replay Handles Partial Writes and Corruption

The core logic lives in `_read_record` (`wal.py:37–56`), and the design uses a **two-layer defense**:

### Layer 1: Short Reads (Partial Writes)

```python
length_data = f.read(4)
if len(length_data) < 4:
    return None                    # can't even read the length prefix

record_length = struct.unpack("<I", length_data)[0]
record_data = f.read(record_length)
if len(record_data) < record_length:
    return None                    # record is truncated
```

If a crash happened mid-write, either the 4-byte length prefix is incomplete, or the record body is shorter than `record_length` declares. In both cases, `_read_record` returns `None` — the same signal as a clean EOF. **The partial record is silently discarded.**

### Layer 2: CRC Mismatch (Corruption)

```python
crc_data = struct.pack("B", op_type_byte) + key + value
expected_crc = zlib.crc32(crc_data) & 0xFFFFFFFF
if crc != expected_crc:
    raise ValueError(f"CRC mismatch at seq {seq_num}")
```

If the bytes are all present but garbled (bit flip, torn write that wrote the length correctly but corrupted the payload), the CRC check catches it and **raises a `ValueError`**.

### How Callers React

The callers of `_read_record` — `_recover_seq_num` (`wal.py:84–94`) and presumably `replay`/`iterate` (in the unshown portion after line 200) — handle these two signals differently but with the same practical effect:

```python
# from _recover_seq_num, wal.py:88–94
while True:
    try:
        rec = _read_record(f)
        if rec is None:
            break              # EOF or partial → stop reading this file
        max_seq = max(max_seq, rec.seq_num)
    except ValueError:
        break                  # CRC error → stop reading this file
```

**Both partial writes and CRC errors terminate reading of the current file.** Any records *after* the corrupted one — even if they're valid — are abandoned. This is a deliberate choice: since records have no random-access index, there's no reliable way to find the next valid record boundary after corruption. It's the same strategy PostgreSQL and LevelDB use.

### What This Means in Practice

The `test_corruption` test (`test_wal.py:47–57`) demonstrates this directly: it writes two records, then overwrites the last 5 bytes of the file with `0xFF`. On replay, only the first record survives — the second is either partially unreadable or fails CRC, and reading stops.

### Sync Modes and Durability

The `_do_sync` method (`wal.py:112–121`) controls when `fsync` actually happens:

| Mode | Behavior | Partial-write window |
|------|----------|---------------------|
| `"sync"` | fsync after every append | Minimal — only the in-flight write |
| `"batch"` | fsync every N writes (`batch_sync_count`) | Up to N records could be lost |
| `"none"` | Never explicit fsync | OS decides; entire buffer could be lost |

In `"batch"` and `"none"` modes, the replay logic becomes more important because more records may be in an intermediate state after a crash.

### A Subtle Gap

The CRC covers `op_type + key + value` but **not** `seq_num`, `key_len`, or `val_len`. If corruption hits `key_len` specifically — making it larger than actual — the read would grab bytes from the next record as part of the key, and the CRC would fail, which is the correct outcome. But if `seq_num` alone is corrupted, the CRC would still pass, and replay would accept a record with a wrong sequence number. This is unlikely to cause data loss but could confuse sequence-number-dependent logic like `truncate` or `replay(after_seq=...)`.

---

## Topics to Explore

- [file] `log-structured-merge-tree/lsm.py` — Shows how the LSM tree uses a simpler WAL class (`lsm.py:13`) as a component, including replay on startup (`lsm.py:230`) and truncation after memtable flush (`lsm.py:314`)
- [function] `write-ahead-log/wal.py:append_batch` — Trace how batched writes achieve atomicity by buffering all records plus a COMMIT marker and writing them in a single `write()` call
- [function] `write-ahead-log/wal.py:truncate` — Understand how old records are pruned: the method rewrites files keeping only records with `seq_num > up_to_seq`, removing files that become empty
- [general] `fsync-durability-guarantees` — Explore what `os.fsync` actually guarantees on different filesystems (ext4, XFS, APFS) and why `flush()` alone is insufficient
- [general] `record-boundary-recovery` — Compare this "stop at first error" approach with alternatives like length-prefixed scanning with magic bytes or block-aligned records (as in LevelDB's log format)

## Beliefs

- `wal-record-length-prefixed` — Every WAL record is prefixed with a 4-byte little-endian length that covers all subsequent fields (CRC through value), enabling the reader to know exactly how many bytes to consume
- `wal-crc-covers-payload-only` — The CRC32 checksum covers `op_type + key + value` but excludes `seq_num`, `record_length`, `key_len`, and `val_len`, meaning sequence number corruption would go undetected
- `wal-corruption-stops-file-scan` — Both partial reads (`None` return) and CRC mismatches (`ValueError`) cause the reader to stop processing the current WAL file, discarding any subsequent records in that file even if they are valid
- `wal-batch-atomicity-via-commit-record` — A batch is only considered complete if its trailing `COMMIT` record is present and passes CRC; incomplete batches (missing commit) are excluded by `replay` which filters on checkpoint/commit boundaries

