# Topic: The on-disk record format (`HEADER_FMT`, CRC, key-length, value-length, payload) is central to tests 4 and 6; understanding the exact byte layout explains the magic offsets

**Date:** 2026-05-29
**Time:** 08:34

# The Bitcask On-Disk Record Format

## The byte layout

Every record in the log-structured hash table is a fixed-size header followed by a variable-size payload. The format is defined at `log-structured-hash-table/bitcask.py:10-11`:

```python
HEADER_FMT = "!III"  # crc32, key_size, value_size
HEADER_SIZE = struct.calcsize(HEADER_FMT)  # = 12 bytes
```

The `!III` format string means: network byte order (big-endian), three unsigned 32-bit integers. Each `I` is 4 bytes, so the header is always exactly **12 bytes**. A single record on disk looks like this:

```
 0         4         8        12        12+K      12+K+V
 ├─────────┼─────────┼────────┼─────────┼─────────┤
 │  CRC32  │ key_len │val_len │  key    │  value  │
 │ 4 bytes │ 4 bytes │4 bytes │ K bytes │ V bytes │
 └─────────┴─────────┴────────┴─────────┴─────────┘
 ◄──────── HEADER ─────────────►◄───── PAYLOAD ────►
        (12 bytes fixed)           (variable)
```

Total record size = `12 + len(key_bytes) + len(value_bytes)`.

## How records are written and read

**Writing** (`bitcask.py:138-145`): The CRC is computed over the **payload only** (key bytes concatenated with value bytes), not over the header. The header is then prepended and the whole thing is written atomically:

```python
payload = key_bytes + value
crc = zlib.crc32(payload) & 0xFFFFFFFF
header = struct.pack(HEADER_FMT, crc, len(key_bytes), len(value))
self._active_file.write(header + payload)
```

**Reading** (`bitcask.py:170-185`): The reader seeks to the offset stored in the in-memory index, reads exactly 12 bytes of header to learn the sizes, reads `key_size + value_size` bytes of payload, then recomputes the CRC and compares:

```python
crc, key_size, value_size = struct.unpack(HEADER_FMT, header)
payload = f.read(key_size + value_size)
expected_crc = zlib.crc32(payload) & 0xFFFFFFFF
if crc != expected_crc:
    raise CorruptionError(...)
```

## Why this explains the "magic offsets" in tests 4 and 6

### Test 4: Segment rotation (`test_bitcask.py:53-58`)

```python
def test_segment_rotation(store):
    for i in range(20):
        store.put(f"key{i}", b"x" * 10)
    assert store.num_segments >= 2
```

The fixture creates a store with `max_segment_size=100`. Each record is:

| Component | Size |
|-----------|------|
| Header | 12 bytes |
| Key (`"key0"` .. `"key9"`) | 4 bytes |
| Value (`b"x" * 10`) | 10 bytes |
| **Total per record** | **26 bytes** |

After 3 records: 78 bytes (under 100). After 4 records: 104 bytes (over 100). So rotation triggers on the 4th write. The threshold of 100 isn't arbitrary — it's chosen so that 12 + 4 + 10 = 26 bytes per record guarantees rotation within a handful of writes. Without knowing the 12-byte header overhead, 100 looks like a magic number; with it, it's obviously ~3.8 records per segment.

### Test 6: Disk usage decreases (`test_bitcask.py:82-91`)

```python
for i in range(10):
    store.put("key", f"value_{i}".encode())  # 10 overwrites = 9 stale
for i in range(10):
    store.put(f"other{i}", b"data")
```

Again `max_segment_size=100`. The first loop writes `"key"` (3 bytes) + `"value_0"` through `"value_9"` (7-8 bytes each), so each record is 12 + 3 + 7 = **22 bytes** (or 23 for two-digit indices). About 4 records fit per segment before rotation. All 10 overwrites of `"key"` create 9 stale records spread across multiple segments. The second loop adds more records with similar sizing. After compaction, only the latest `"key"` entry and the 10 `"other*"` entries survive — the 9 stale records (each ~22 bytes = ~198 bytes of dead data) are eliminated, explaining the measurable drop in `total_disk_usage`.

### Test 9: CRC corruption (`test_bitcask.py:131`)

The "magic offset" here is explicit:

```python
f.seek(offset + HEADER_SIZE + 2)  # inside the key/value payload
```

`offset + HEADER_SIZE` lands exactly at the first byte of the payload (start of the key). Adding 2 puts you inside the key/value data. Since the CRC covers the payload but not the header, flipping any byte past `offset + 12` invalidates the checksum while leaving the header (and its recorded CRC) untouched — guaranteeing a `CorruptionError` on the next `get()`.

## Key design choice: CRC covers payload only

The CRC is computed over `key_bytes + value_bytes` but **not** the header fields (`key_size`, `value_size`). This means a corrupted size field wouldn't be caught by the CRC — instead, it would cause a wrong-length read, which would almost certainly produce a CRC mismatch anyway (reading the wrong bytes). This is a pragmatic simplification common in Bitcask-style implementations: the header is small and fixed, so corruption there manifests as garbage sizes that either fail to read or fail the CRC check on the misaligned payload.

---

## Topics to Explore

- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — How recovery walks the byte layout sequentially, stopping at the first CRC mismatch to handle partial writes
- [function] `log-structured-hash-table/bitcask.py:compact` — How compaction rewrites records using the same format but with fresh CRCs (`new_crc` at line 275-276)
- [file] `log-structured-hash-table/tester_test_bitcask.py` — An alternative test suite that manually constructs records with `struct.pack(HEADER_FMT, ...)` (line 146), showing the format from the test-author's perspective
- [general] `tombstone-encoding` — How deletes use `TOMBSTONE = b"__BITCASK_TOMBSTONE__"` as a sentinel value within the same record format, and the implications for key-size/value-size calculations
- [file] `hash-index-storage/bitcask.py` — A second Bitcask implementation with a different header format (`"<dII"` — timestamp + sizes, no CRC), showing an alternative design tradeoff

## Beliefs

- `bitcask-header-is-12-bytes` — `HEADER_FMT = "!III"` produces a fixed 12-byte header (three big-endian uint32s: CRC, key_size, value_size) at `log-structured-hash-table/bitcask.py:10-11`
- `crc-covers-payload-not-header` — The CRC32 is computed over `key_bytes + value_bytes` only; header fields (sizes) are not integrity-checked by the CRC
- `record-size-formula` — Total on-disk record size is always `HEADER_SIZE + len(key_bytes) + len(value_bytes)`, i.e., `12 + K + V` bytes
- `segment-rotation-threshold-depends-on-record-size` — Tests using `max_segment_size=100` rely on records being ~22-27 bytes each, guaranteeing rotation after 3-4 writes
- `corruption-detection-on-read-not-write` — CRC validation happens only during `get()` and `_scan_segment()`; writes compute and store the CRC but never verify it back

