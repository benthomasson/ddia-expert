# Topic: Compare the CRC boundary choices in bitcask, b-tree WAL, and this WAL to see if the payload-only pattern is a deliberate project convention or independent coincidence

**Date:** 2026-05-29
**Time:** 06:28

# CRC Boundary Choices Across Three Implementations

## The Three CRC Strategies

Each implementation that uses CRC makes a distinct choice about what data the checksum covers:

### 1. Log-Structured Bitcask (`log-structured-hash-table/bitcask.py:155-157`)

```python
payload = key_bytes + value
crc = zlib.crc32(payload) & 0xFFFFFFFF
header = struct.pack(HEADER_FMT, crc, len(key_bytes), len(value))
```

**CRC covers: `key + value`** (pure payload). The header fields (`crc`, `key_size`, `value_size`) are excluded. The CRC sits *inside* the header, protecting only the bytes that follow it.

### 2. B-Tree WAL (`b-tree-storage-engine/btree.py:134-137`)

```python
header = struct.pack(self.ENTRY_HEADER, self._seq, page_num, len(page_data))
checksum = struct.pack('>I', self._checksum(page_data))
self._f.write(header + page_data + checksum)
```

**CRC covers: `page_data`** (pure payload). The header fields (`seq`, `page_num`, `data_len`) are excluded. Here the CRC is a *trailer* rather than a header field, but the boundary is the same — only the variable-length data blob.

### 3. Standalone WAL (`write-ahead-log/wal.py:30-32`)

```python
crc_data = struct.pack("B", op_type_byte) + key + value
crc = zlib.crc32(crc_data) & 0xFFFFFFFF
```

**CRC covers: `op_type + key + value`** — payload *plus one header field*. The `seq_num`, `record_length`, `key_len`, and `value_len` are all excluded, but `op_type_byte` is included in both the CRC input and the serialized header.

### 4. Hash-Index Bitcask (`hash-index-storage/bitcask.py`)

**No CRC at all.** The header format `<dII` contains `timestamp`, `key_size`, `value_size` with no integrity field. This implementation relies entirely on structural validation (successful parsing) to detect corruption.

## Verdict: Independent Coincidence, Not Convention

Three pieces of evidence point to coincidence rather than deliberate coordination:

1. **The boundaries aren't actually identical.** The standalone WAL breaks the payload-only pattern by pulling `op_type_byte` into the CRC input (`wal.py:30`). If this were a project convention, you'd expect a shared utility or at least a consistent rule. Instead, one implementation includes a header field and the others don't.

2. **The fourth implementation has no CRC at all.** The hash-index bitcask (`hash-index-storage/bitcask.py`) omits integrity checking entirely. A project-wide convention would presumably apply to all storage engines, not three of four.

3. **The CRC placement differs.** The log-structured bitcask puts CRC in the header (`!III` — crc first). The b-tree WAL puts it as a trailer (after `page_data`). The standalone WAL puts it second in the header (`<IIQBi` — length, then crc). These are structurally different record formats that happen to share a pragmatic instinct.

**The shared instinct** is what makes this look like a convention at first glance: all three exclude *framing metadata* (lengths, sequence numbers, offsets) from the CRC. This makes practical sense — if a length field is corrupted, the read will either grab the wrong number of bytes (producing a CRC mismatch anyway) or fail structurally. Checksumming lengths provides little additional protection over checksumming the payload alone. This is a well-known pattern in storage format design (e.g., LevelDB's block checksums cover block data + type byte but not the length framing), and each implementation likely arrived at it independently from the same reasoning.

The standalone WAL's inclusion of `op_type_byte` is the telling detail: it protects against the scenario where the framing is intact but the operation type is silently flipped (e.g., a PUT becomes a DELETE). This is a *semantic* concern that doesn't apply to the b-tree WAL (which logs raw pages with no operation semantics) or the bitcask (where key+value already encodes the full intent). Each author chose the CRC boundary that matched their corruption threat model, not a shared rule.

## Topics to Explore

- [function] `write-ahead-log/wal.py:_encode_record` — Why `op_type_byte` is included in the CRC but `seq_num` is not, and what corruption scenarios each choice defends against
- [function] `b-tree-storage-engine/btree.py:recover` — How the b-tree WAL handles partial writes and whether trailing CRC placement interacts differently with torn-write recovery than header CRC
- [general] `hash-index-bitcask-no-crc` — Whether the absence of CRC in `hash-index-storage/bitcask.py` is a deliberate simplification or an oversight, and what happens on silent corruption
- [general] `crc32-vs-xxhash-choice` — All three use `zlib.crc32`; whether this is adequate for the file sizes involved or whether a faster/stronger hash would change the design
- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — How CRC failure at line 101 triggers a `break` (stop scanning) rather than skipping the record, and the implications for data after a corrupted record

## Beliefs

- `payload-only-crc-is-coincidence` — The three CRC implementations independently arrived at similar payload-centric boundaries rather than following a project convention; the standalone WAL's inclusion of op_type in its CRC input is the clearest evidence of independent design
- `hash-index-bitcask-has-no-integrity-check` — The hash-index-storage bitcask (`hash-index-storage/bitcask.py`) has no CRC or checksum field, relying entirely on structural parse success to detect corruption
- `wal-crc-covers-op-type-unlike-others` — The standalone WAL (`write-ahead-log/wal.py:30`) uniquely includes `op_type_byte` in its CRC input, protecting against silent operation-type corruption that the other implementations leave undetected
- `btree-wal-uses-trailer-crc` — The b-tree WAL places its checksum as a trailer after `page_data` (`btree.py:136`), while the other implementations embed CRC in the record header — a structural difference that affects torn-write detection behavior

