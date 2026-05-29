# Topic: Consider what a v2 record format would look like: CRC over the full header, a magic byte for format versioning, and forward-compatible length-prefixed fields

**Date:** 2026-05-29
**Time:** 06:29

# Designing a v2 WAL Record Format

## The Current v1 Format and Its Weaknesses

The WAL in `write-ahead-log/wal.py:29-34` uses this record layout:

```
[record_length:4][crc:4][seq_num:8][op_type:1][key_len:4] [key] [val_len:4] [value]
```

Three problems stand out:

**1. The CRC doesn't cover the header.** At line 30, only the op type, key, and value are checksummed:

```python
crc_data = struct.pack("B", op_type_byte) + key + value
```

The `seq_num`, `record_length`, and `key_len` fields are **not** protected. A bit flip in `seq_num` would silently produce a valid-looking record with the wrong sequence number. A corrupted `key_len` would cause the reader to slice the key/value boundary in the wrong place — the CRC would then fail, but the error message at line 56 would blame the wrong thing (`CRC mismatch at seq {seq_num}`, where `seq_num` itself might be garbage).

**2. No format version marker.** There's no way for a reader to distinguish v1 records from a hypothetical v2. If you change the layout, old readers will silently misparse new files. Contrast this with the SSTable implementation in `sstable-and-compaction/sstable.py:10-16`, which already does it right:

```python
MAGIC = b"SSTB"
VERSION = 1
HEADER_FMT = ">4sHI"  # magic(4) + version(2) + entry_count(4)
```

The SSTable reader validates the magic at line 119 (`assert magic == MAGIC`) before touching any data. The WAL has no equivalent guard.

**3. No forward compatibility.** Every field is positionally encoded. Adding a new field (say, a transaction ID or a compression flag) requires changing the struct format string, which breaks all existing readers. There's no mechanism to skip unknown fields.

## What v2 Would Look Like

Here's the concrete design, building on patterns already present in the codebase:

### Record-level framing

```
[magic:1][version:1][record_length:4][header_crc:4] [header fields...] [key] [value] [payload_crc:4]
```

**Magic byte** (`0xAB` or similar): Lets the reader distinguish WAL records from garbage, partial writes, or other file formats. The SSTable uses a 4-byte magic (`b"SSTB"`) but a single byte suffices per-record since we don't need human readability — we need a fast rejection test during recovery scans.

**Version byte**: Tells the reader which header layout follows. `0x01` = v1 compatibility shim, `0x02` = the new format. A reader seeing an unknown version can skip `record_length` bytes and continue recovery — degraded but not dead.

**Header CRC**: A CRC32 over `[magic][version][record_length]` and all header fields. This is the key fix. Currently (`wal.py:30-31`), the CRC covers only the payload, so header corruption goes undetected. In v2, a corrupted `seq_num` or `key_len` would be caught before the reader even attempts to parse the key/value data.

**Payload CRC**: A separate CRC32 over key + value bytes. Splitting the CRCs lets you detect *which part* is corrupt — header damage vs. payload damage require different recovery strategies.

### Length-prefixed fields for forward compatibility

Instead of a fixed struct like `<IIQBi` (`wal.py:33`), each header field becomes a TLV (type-length-value) triple:

```
[field_tag:1][field_len:2][field_data:variable]
```

Known tags for v2:
- `0x01` = seq_num (8 bytes)
- `0x02` = op_type (1 byte)
- `0x03` = key_len (4 bytes)
- `0x04` = val_len (4 bytes)

A v3 reader encountering an unknown tag (say `0x05` = transaction_id) skips `field_len` bytes and keeps going. This is exactly how protocol buffers achieve wire compatibility, and it's why the Bitcask format in `hash-index-storage/bitcask.py:10` — which uses a fixed `HEADER_FORMAT = "<dII"` — would also benefit from this pattern.

The tradeoff: TLV adds ~3 bytes of overhead per field (5 fields = 15 bytes). For a WAL that fsyncs every record (`wal.py:120-125`), those extra bytes are negligible compared to the disk flush cost. The LSM WAL in `log-structured-merge-tree/lsm.py:20-25` uses bare length-prefixed writes with no tags at all — even simpler but even less evolvable.

### How recovery changes

The current recovery loop in `_read_record` (`wal.py:37-58`) does this:

1. Read 4 bytes for `record_length`
2. Read `record_length` bytes
3. Parse header fields
4. Verify CRC over payload only

v2 recovery would:

1. Scan for magic byte (resync after corruption)
2. Read version + record_length
3. Verify header CRC — if it fails, skip `record_length` bytes and scan for next magic
4. Parse TLV fields, skipping unknowns
5. Read key + value
6. Verify payload CRC

The magic-byte scan is the biggest operational improvement. Right now, a single corrupted `record_length` at line 40-43 causes `_read_record` to read the wrong number of bytes, and every subsequent record is misframed. With a magic byte, the reader can re-synchronize by scanning forward until it finds the next `0xAB`.

### A concrete struct sketch

```python
RECORD_MAGIC = 0xAB
RECORD_VERSION = 2

def _encode_record_v2(seq_num, op_type_byte, key, value):
    # Build TLV header fields
    fields = bytearray()
    fields += struct.pack("<BH", 0x01, 8) + struct.pack("<Q", seq_num)
    fields += struct.pack("<BH", 0x02, 1) + struct.pack("B", op_type_byte)
    fields += struct.pack("<BH", 0x03, 4) + struct.pack("<I", len(key))
    fields += struct.pack("<BH", 0x04, 4) + struct.pack("<I", len(value))

    payload = key + value
    record_length = len(fields) + len(payload) + 4  # +4 for payload CRC

    # Header: magic + version + length
    preamble = struct.pack("BB", RECORD_MAGIC, RECORD_VERSION)
    preamble += struct.pack("<I", record_length)

    header_crc = zlib.crc32(preamble + fields) & 0xFFFFFFFF
    payload_crc = zlib.crc32(payload) & 0xFFFFFFFF

    return (preamble + struct.pack("<I", header_crc) +
            bytes(fields) + payload +
            struct.pack("<I", payload_crc))
```

## Patterns Already in the Codebase

The SSTable format (`sstable-and-compaction/sstable.py:10-20`) is the closest existing example of good practice: it has magic bytes, a version field, and structured footers. The WAL and Bitcask formats are the ones most in need of this upgrade — they're the components that must survive crashes and be readable across software versions.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_read_record` — Trace how a single corrupted record_length cascades into total read failure, motivating the magic-byte resync design
- [file] `sstable-and-compaction/sstable.py` — The SSTable already implements magic + version + footer index; compare its evolution-readiness to the WAL's
- [function] `hash-index-storage/bitcask.py:_write_record` — Bitcask has the same fixed-header fragility as the WAL; a v2 format would apply identically here
- [general] `crc-coverage-audit` — Audit all struct.pack/CRC sites across the codebase (87 matches found) to identify which components protect only payload vs. full-record integrity
- [file] `b-tree-storage-engine/btree.py` — The B-tree WAL (line 120-135) uses its own entry format with checksums; compare its integrity guarantees to the standalone WAL

## Beliefs

- `wal-crc-excludes-header` — The WAL CRC at `wal.py:30-31` covers only `op_type + key + value`, leaving `seq_num`, `record_length`, and `key_len` unprotected against bit flips
- `sstable-has-magic-and-version` — The SSTable format (`sstable.py:10-11`) uses a 4-byte magic `b"SSTB"` and a 2-byte version field, making it the only binary format in the repo with format versioning
- `wal-no-resync-after-corruption` — A corrupted `record_length` in the WAL causes `_read_record` to misframe all subsequent records; there is no scan-forward recovery mechanism
- `bitcask-no-integrity-check` — The Bitcask store (`bitcask.py:10`) uses a fixed `<dII` header with no CRC at all — records have no corruption detection

