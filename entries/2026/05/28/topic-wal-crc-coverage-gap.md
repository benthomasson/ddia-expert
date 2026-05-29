# Topic: The CRC excludes `seq_num` and `record_length` from integrity checking — worth understanding whether this is an intentional design choice or a gap

**Date:** 2026-05-28
**Time:** 18:49

## CRC Scope in the WAL: Intentional Framing vs. Silent `seq_num` Corruption

The CRC in this WAL covers **only the payload** — `op_type_byte + key + value` — and excludes two header fields: `record_length` and `seq_num`. These are two different situations, and the answer is split: one exclusion is defensible, the other is a real gap.

### What the CRC covers

In `_encode_record` (`wal.py:30-31`):
```python
crc_data = struct.pack("B", op_type_byte) + key + value
crc = zlib.crc32(crc_data) & 0xFFFFFFFF
```

The verification in `_read_record` (`wal.py:53-55`) mirrors this exactly — it recomputes the CRC over the same three fields and rejects on mismatch.

### `record_length` exclusion: defensible

`record_length` is a **framing field** — it's read first (`wal.py:39-42`) to determine how many bytes to consume for the rest of the record. If `record_length` is corrupted:

- **Too small**: You'd read a truncated record. The key/value bytes would be wrong, and the CRC check would almost certainly fail.
- **Too large**: You'd read past the record boundary into the next record's bytes (or hit EOF and return `None` at line 44). Either way, the CRC over garbage bytes would fail.

This is a standard pattern. The framing field is *implicitly* validated because any corruption to it causes the payload CRC to fail. Including it in the CRC would be belt-and-suspenders — not wrong, but not necessary for correctness.

### `seq_num` exclusion: a real gap

`seq_num` is **not a framing field** — it doesn't affect how the record is parsed. A bit flip in `seq_num` produces a record that parses correctly, passes CRC validation, and is silently accepted with a wrong sequence number.

This matters because `seq_num` drives three critical control paths:

1. **Recovery ordering** (`_recover_seq_num`, `wal.py:85-96`): The WAL scans all records to find `max(seq_num)` and resumes numbering from there. A corrupted `seq_num` inflated to a huge value would create a gap in the sequence space.

2. **Truncation** (`truncate`, `wal.py:196`): Records are kept or discarded based on `rec.seq_num > up_to_seq`. A corrupted `seq_num` could cause a record to survive truncation (if inflated) or be incorrectly discarded (if deflated).

3. **Replay filtering** (`replay`, `wal.py:226`): `rec.seq_num <= after_seq` determines which records to skip. A corrupted sequence number could cause committed data to be skipped during crash recovery.

### Comparison with other implementations in this repo

The same payload-only CRC pattern appears in the Bitcask implementation (`bitcask.py:142-143`), where CRC covers `key + value` but not `key_size` or `value_size`. The B-tree WAL (`btree.py:133,175-176`) checksums `page_data` but not `seq`, `page_num`, or `data_len`. This is consistent — all three implementations treat metadata as outside the CRC boundary — but the WAL's `seq_num` has the most dangerous failure mode because it controls replay and truncation correctness.

### Verdict

The `record_length` exclusion is an **intentional design choice** (standard framing pattern). The `seq_num` exclusion is a **gap** — including `seq_num` in the CRC input would cost nothing and would protect against silent ordering corruption during recovery. The fix would be a one-line change to line 30:

```python
crc_data = struct.pack("<QB", seq_num, op_type_byte) + key + value
```

with the corresponding change in `_read_record` at line 53.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_record` — Trace how a corrupted `record_length` propagates: does every corruption path actually land on a CRC failure, or are there edge cases where it silently misframes?
- [function] `write-ahead-log/wal.py:replay` — Understand the committed-transaction replay logic and how `seq_num` ordering interacts with the `pending` buffer to determine atomicity
- [general] `crc-scope-comparison-across-implementations` — Compare the CRC boundary choices in bitcask, b-tree WAL, and this WAL to see if the payload-only pattern is a deliberate project convention or independent coincidence
- [file] `b-tree-storage-engine/btree.py` — The B-tree WAL checksums `page_data` only; examine whether its `seq` and `page_num` fields are vulnerable to the same silent-corruption class
- [general] `wal-record-format-evolution` — Consider what a v2 record format would look like: CRC over the full header, a magic byte for format versioning, and forward-compatible length-prefixed fields

## Beliefs

- `wal-crc-covers-payload-only` — The WAL CRC at `wal.py:30-31` checksums `op_type_byte + key + value` only; `seq_num` and `record_length` are excluded from integrity checking
- `seq-num-corruption-is-silent` — A bit flip in a WAL record's `seq_num` field will pass CRC validation and be accepted with the wrong sequence number, potentially corrupting replay, truncation, and recovery ordering
- `record-length-implicitly-validated` — Corruption of `record_length` causes `_read_record` to consume wrong bytes, which fails the payload CRC check indirectly, making explicit inclusion in the CRC unnecessary
- `all-three-implementations-use-payload-only-crc` — The WAL, Bitcask, and B-tree storage engine all compute CRC over data payloads only, excluding their respective header/framing metadata fields

