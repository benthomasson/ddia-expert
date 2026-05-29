# Topic: How CRC32 in `_read_record` (line 57-59) interacts with OS write atomicity to detect torn writes

**Date:** 2026-05-29
**Time:** 06:33

## CRC32 and Torn Write Detection in `_read_record`

### The Two-Layer Defense

`_read_record` (`write-ahead-log/wal.py:37`) uses two distinct mechanisms to handle incomplete writes, and they catch different failure modes:

**Layer 1 — Length-prefix framing (lines 39–44):** Every record is preceded by a 4-byte `record_length`. On read, if fewer than 4 bytes are available (line 41) or the body is shorter than `record_length` (line 44), the function returns `None` silently. This catches the common torn-write case: a crash interrupts `write()` mid-record, so the file simply ends with a truncated record. The reader treats this as EOF — no error, just "this record didn't land."

**Layer 2 — CRC32 verification (lines 53–56):** If the length prefix *and* the full body are present but the data is wrong, the CRC catches it. The check recomputes `zlib.crc32` over `op_type_byte + key + value` and compares against the stored CRC at the front of the record. A mismatch raises `ValueError` — this is treated as corruption, not a benign truncation.

### Why Both Layers Are Needed

POSIX `write()` is **not atomic** for arbitrarily sized buffers. The OS and disk can commit data in sector-sized chunks (typically 512B or 4KB). A crash during a multi-sector write produces a record where:

- The **length prefix and early fields are valid** (first sector committed)
- The **tail of the record contains stale data** from whatever was on disk before (later sectors not committed)

In this scenario, layer 1 sees a complete record (the length and body sizes match), so it doesn't trigger. But layer 2 catches it — the CRC was computed from the intended payload in `_encode_record` (line 31), and the stale tail bytes produce a different checksum.

### What the CRC Covers (and Doesn't)

Looking at `_encode_record` (line 30–31):

```python
crc_data = struct.pack("B", op_type_byte) + key + value
crc = zlib.crc32(crc_data) & 0xFFFFFFFF
```

The CRC covers **op_type, key, and value** — the semantic payload. It does **not** cover `seq_num`, `key_len`, or `val_len`. This is a pragmatic choice: if `key_len` or `val_len` are corrupted by a torn write, the reader slices the wrong bytes for key/value, which overwhelmingly produces a CRC mismatch anyway. However, a torn write that corrupts **only** `seq_num` (bytes 5–12 of the record body) would go undetected — the record would be accepted with a wrong sequence number.

### The Recovery Contract

The design establishes a clear contract: `_read_record` returns `None` for truncation (benign, expected at tail after crash) and raises `ValueError` for corruption (torn write that produced a plausible-length but wrong-content record). Recovery code upstream can use this distinction — skip tail truncation, but flag or halt on CRC errors.

### Notable Gap

The grep for `torn|corrupt|truncat|partial` in the WAL tests returned zero matches. There are no tests exercising the torn-write detection path, so the CRC-mismatch behavior is untested at the WAL level (though `b-tree-storage-engine/test_btree.py:257` does test CRC corruption for the B-tree).

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:_encode_record` — How the record binary format is assembled and where `fsync`/`fdatasync` are called relative to the write, which determines what "committed" means
- [function] `write-ahead-log/wal.py:_recover_seq_num` — How recovery walks the log, and whether it stops on CRC error or skips past it — this determines the practical effect of a torn write
- [file] `log-structured-hash-table/bitcask.py` — Uses the same CRC32 pattern but with a different header format (`!III` — big-endian, CRC covers the full payload); comparing the two designs reveals different trade-offs
- [general] `wal-sync-modes` — The `sync_mode` parameter and `batch_sync_count` determine how often `fsync` is called, which directly controls the window of vulnerability to torn writes
- [general] `seq-num-integrity-gap` — Whether the missing CRC coverage of `seq_num` can cause silent mis-ordering during recovery, and whether adding `seq_num` to the CRC input is a worthwhile fix

---

## Beliefs

- `wal-crc-does-not-cover-seqnum` — CRC32 in `_read_record` is computed over `op_type + key + value` only; a torn write corrupting only `seq_num` passes validation silently
- `wal-truncation-vs-corruption-distinction` — `_read_record` returns `None` for short reads (truncation) but raises `ValueError` for CRC mismatch (corruption), establishing two distinct failure modes for callers
- `wal-record-length-prefix-guards-short-writes` — The 4-byte length prefix at the start of each record allows `_read_record` to detect incomplete writes without reaching the CRC check
- `wal-no-torn-write-tests` — The WAL module has no tests for truncated records, CRC mismatches, or partial writes, unlike the B-tree module which tests CRC corruption explicitly

