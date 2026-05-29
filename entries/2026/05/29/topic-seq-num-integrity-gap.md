# Topic: Whether the missing CRC coverage of `seq_num` can cause silent mis-ordering during recovery, and whether adding `seq_num` to the CRC input is a worthwhile fix

**Date:** 2026-05-29
**Time:** 08:20

Now I have a thorough understanding. Let me write the explanation.

---

# Can Missing CRC Coverage of `seq_num` Cause Silent Mis-Ordering During Recovery?

**Yes, it can.** The gap is real, the failure modes are concrete, and the fix is trivial. Here's the full picture.

## The CRC Boundary

In `_encode_record` (`wal.py:30-31`), the CRC32 is computed over three fields:

```python
crc_data = struct.pack("B", op_type_byte) + key + value
crc = zlib.crc32(crc_data) & 0xFFFFFFFF
```

The on-disk record format is:

```
[record_length:4][crc:4][seq_num:8][op_type:1][key_len:4][key:N][val_len:4][value:M]
```

The `seq_num` (8 bytes, unsigned 64-bit) sits between the CRC and the op_type. It is **written to disk but never integrity-checked**. The corresponding verification in `_read_record` (`wal.py:53-55`) recomputes the CRC over the same `op_type + key + value` triple and compares — so a bit flip in the `seq_num` field produces a record that parses cleanly, passes CRC validation, and is silently accepted with a corrupted sequence number.

## Three Concrete Failure Modes

The `seq_num` field drives three critical control paths, each with a distinct corruption scenario:

### 1. Recovery Ordering via `_recover_seq_num`

`_recover_seq_num` (called once during `__init__`) scans all WAL files to find `max(seq_num)` and resumes numbering from `max + 1`. If a bit flip inflates a `seq_num` to a huge value — say `2^60` — the WAL will set `self._seq_num = 2^60` and start issuing sequence numbers from there. This creates a massive gap in the sequence space. While it doesn't directly lose data, it can interact badly with:

- Truncation logic that uses sequence ranges to decide what to keep
- Any external system that expects monotonically increasing, gap-free sequence numbers
- Debugging, where a sudden jump to astronomical sequence numbers obscures the real history

### 2. Truncation via `truncate`

`truncate(up_to_seq)` keeps records where `rec.seq_num > up_to_seq` and discards the rest. A corrupted `seq_num` has two failure modes:

- **Inflated** (bit flip increases the value): The record survives truncation when it should have been removed. This wastes space and could cause stale data to reappear during replay.
- **Deflated** (bit flip decreases the value): The record is incorrectly discarded. **This is data loss** — a committed, valid record is silently dropped because its sequence number fell below the truncation watermark.

### 3. Replay Filtering via `replay`

`replay(after_seq)` skips records where `rec.seq_num <= after_seq`. A corrupted `seq_num` can:

- **Deflated**: Cause a record to be skipped during crash recovery (it looks like it was already applied). The consumer never sees the operation, and the database silently diverges from the intended state.
- **Inflated**: Cause a record to be re-applied when it shouldn't be. If the operation is idempotent (PUT), this is benign. If it's not (consider a counter increment), it's a subtle correctness bug.

## Why `record_length` Exclusion Is Fine But `seq_num` Is Not

The `record_length` field is a **framing field** — `_read_record` reads it first (`wal.py:39-42`) to determine how many bytes to consume. If `record_length` is corrupted:

- **Too small**: Truncated payload → CRC mismatch
- **Too large**: Reads into next record or hits EOF → either CRC mismatch or `None` return

The framing field is **implicitly validated** because any corruption cascades into a CRC failure. This is a standard pattern.

`seq_num` is fundamentally different: it's a **semantic field**, not a framing field. It doesn't affect how the record is parsed — only what the record *means*. A corrupted `seq_num` produces a structurally valid record that passes every integrity check the WAL performs.

## Is the Fix Worthwhile?

**Unambiguously yes.** The fix is a one-line change in `_encode_record` (`wal.py:30`):

```python
# Before:
crc_data = struct.pack("B", op_type_byte) + key + value

# After:
crc_data = struct.pack("<QB", seq_num, op_type_byte) + key + value
```

With the corresponding change in `_read_record` (`wal.py:53`):

```python
# Before:
computed_crc = zlib.crc32(struct.pack("B", op_type) + key + value) & 0xFFFFFFFF

# After:
computed_crc = zlib.crc32(struct.pack("<QB", seq_num, op_type) + key + value) & 0xFFFFFFFF
```

**Cost**: Zero runtime overhead — CRC32 over 9 additional bytes is negligible. **Benefit**: Turns silent data corruption into a detected CRC failure, which the existing error handling already manages correctly (`_read_record` raises `ValueError`, callers stop or skip).

The only consideration is **format compatibility**: existing WAL files would fail CRC validation after the change. For a reference implementation this is fine (no production data at stake). A production system would need a version byte in the record header and dual-path CRC logic during migration.

## Cross-Implementation Context

The same payload-only CRC pattern appears in two other implementations in this repo:

- **Bitcask** (`bitcask.py:155-157`): CRC covers `key + value` only, excluding `key_size` and `value_size`
- **B-tree WAL** (`btree.py:134-137`): CRC covers `page_data` only, excluding `seq`, `page_num`, and `data_len`

As documented in the CRC scope comparison entry, these are **independent coincidences**, not a project convention — the three implementations use different CRC placements (header vs. trailer), different fields, and one implementation (hash-index bitcask) has no CRC at all. The standalone WAL's `seq_num` exclusion is the most dangerous of the three because `seq_num` drives recovery, truncation, and replay correctness.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_all_records` — The corruption-stopping iterator that terminates the entire replay on CRC failure; understanding its `break` vs `return` semantics clarifies what happens when a corrupted `seq_num` fix causes a new CRC mismatch on old data
- [function] `write-ahead-log/wal.py:append_batch` — How batch atomicity interacts with sequence numbering; a corrupted `seq_num` on a COMMIT record could cause replay to skip or re-apply an entire batch
- [general] `wal-record-format-evolution` — What a v2 record format would look like: CRC over the full header, a magic/version byte for forward compatibility, and migration strategy for existing WAL files
- [function] `b-tree-storage-engine/btree.py:recover` — The B-tree WAL has the same `seq` exclusion from CRC; compare whether its recovery logic is equally vulnerable to silent mis-ordering
- [file] `write-ahead-log/test_wal.py` — Tests exercise CRC corruption and torn writes but likely don't cover `seq_num`-specific corruption; a new test injecting a bit flip into the `seq_num` bytes (offset 8-15 of the record body) would validate the fix

## Beliefs

- `seq-num-corruption-causes-silent-data-loss` — A bit flip in a WAL record's `seq_num` can cause `truncate` to incorrectly discard committed records (if deflated) or `replay` to skip committed operations (if deflated below the `after_seq` watermark), with no error raised
- `seq-num-crc-fix-is-zero-cost` — Adding `seq_num` to the CRC input requires changing two lines (`_encode_record:30` and `_read_record:53`) with no measurable runtime overhead, since CRC32 over 9 extra bytes is negligible
- `record-length-corruption-is-implicitly-caught` — Unlike `seq_num`, corruption of `record_length` cascades into a CRC mismatch because it changes which bytes `_read_record` consumes, making explicit CRC coverage unnecessary
- `wal-format-change-breaks-compatibility` — Changing the CRC input in `_encode_record` invalidates all existing WAL files; a production deployment would require a version byte and dual-path CRC verification during migration

