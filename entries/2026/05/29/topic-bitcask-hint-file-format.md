# Topic: The original Bitcask paper's hint file design and how this implementation diverges (e.g., no CRC, variable-length key encoding)

**Date:** 2026-05-29
**Time:** 08:44

# Bitcask Hint Files: Paper Design vs. Implementation

## What the Original Paper Specifies

The Bitcask paper (Sheehy & Smith, 2010) defines hint files as an optimization for startup. Without them, rebuilding the in-memory keydir requires scanning every data file record — reading keys and values just to populate a `(file_id, offset, size, timestamp)` index. Hint files eliminate this by storing a compact summary of each record alongside the key, so only hint files need to be read on recovery.

The paper's hint entry format is:

```
| tstamp | ksz | value_sz | value_pos | key |
```

Where `tstamp` is a timestamp, `ksz` is the key size, `value_sz` is the value size, `value_pos` is the byte offset of the record in the data file, and `key` is the actual key bytes. Critically, each hint entry mirrors the data record's header fields — everything needed to reconstruct a keydir entry without touching the data file. The paper also specifies that data records carry a CRC for integrity, and production implementations (like the Erlang Bitcask) extend CRC protection to hint files as well.

## Two Implementations, Two Divergences

This repo has two Bitcask implementations that handle hint files quite differently.

### `hash-index-storage/bitcask.py` — Rich but CRC-less hints

This implementation's hint format (`bitcask.py:13`) is:

```python
HINT_FORMAT = "<IQId"  # file_id(uint32), offset(uint64), size(uint32), timestamp(double)
```

Each hint entry is written as this fixed header followed by a variable-length key (`_write_hint_file`, line 157–165):

```python
f.write(struct.pack(HINT_FORMAT, entry.file_id, entry.offset,
                    entry.size, entry.timestamp))
f.write(struct.pack("<I", len(key_bytes)))
f.write(key_bytes)
```

And read back the same way (`_load_hint_file`, line 141–154):

```python
fid, offset, size, ts = struct.unpack(HINT_FORMAT, data[pos:pos + HINT_HEADER_SIZE])
pos += HINT_HEADER_SIZE
key_size = struct.unpack("<I", data[pos:pos + 4])[0]
pos += 4
key = data[pos:pos + key_size].decode("utf-8")
```

**Divergences from the paper:**

1. **No CRC on hint entries.** The data record header (`HEADER_FORMAT = "<dII"`, line 10) also lacks a CRC — it stores only `(timestamp, key_size, value_size)`. A corrupted hint file will silently load bad index entries. The paper's data records include CRC; production hint files extend that protection.

2. **`file_id` stored explicitly in the hint.** The paper's design assumes one hint file per data file, so the file_id is implicit (it's the hint file's own name). This implementation writes `file_id` into each hint entry (`HINT_FORMAT` includes `I` for `file_id`), which is redundant for non-compacted files but necessary during compaction when merged output may span multiple files.

3. **No `value_sz` in hint entries.** The paper includes `value_sz` so the keydir can serve `value_size` without a disk seek. This implementation stores `size` (total record size) instead, which works for reads — you know how many bytes to fetch — but loses the ability to report value size without parsing the record header.

4. **Variable-length key encoding uses a separate length prefix.** The key size is encoded as a 4-byte `<I` *after* the fixed hint header, rather than being part of the fixed header itself. The paper embeds `ksz` in the fixed-width header. This is a minor structural difference but means the hint format isn't a single `struct.unpack` call per entry.

### `log-structured-hash-table/bitcask.py` — Minimal hints with CRC on data

This implementation takes a more stripped-down approach. The hint format (`bitcask.py:12`) is:

```python
HINT_ENTRY_FMT = "!II"  # key_size, offset
```

That's it — just 8 bytes of fixed header per entry, followed by the key bytes (`_load_hint_file`, line 108–120):

```python
key_size, offset = struct.unpack(HINT_ENTRY_FMT, header)
key_bytes = f.read(key_size)
```

**Divergences from the paper:**

1. **No timestamp, no value_sz, no file_id in hints.** The hint stores only `(key_size, offset)`. This means the keydir can't know the timestamp without reading the data record, which breaks the paper's invariant that the keydir entry is fully reconstructable from hints alone. The index here stores `(seg_path, offset)` — no timestamp, no size.

2. **CRC on data records but not on hints.** The data record header (`HEADER_FMT = "!III"`, line 10) includes a CRC32 and the implementation validates it on every read (`get`, line 178–183) and during segment scanning (`_scan_segment`, line 99–100). But hint files have zero integrity protection — a bit flip in a hint offset silently points reads to the wrong location.

3. **Hint files don't record tombstones.** During `_load_hint_file` (line 108–120), every entry is blindly added to the index. There's no mechanism to represent a deleted key in the hint file. If a segment contains a tombstone, the hint file will point to it, and the `get` path will return the tombstone sentinel. This is handled by processing segments in order (oldest first) during recovery, so a later segment's live entry overwrites an earlier tombstone — but only if the key was re-written after deletion.

## Summary Table

| Feature | Bitcask Paper | `hash-index-storage` | `log-structured-hash-table` |
|---|---|---|---|
| CRC on data records | Yes | **No** | Yes |
| CRC on hint entries | Yes (in production) | **No** | **No** |
| Hint fields | tstamp, ksz, value_sz, value_pos, key | file_id, offset, size, timestamp, key_size, key | key_size, offset, key |
| Keydir fully from hints | Yes | Yes | **No** (missing timestamp, size) |
| Tombstone handling in hints | Omitted from hints | Omitted (filtered during compaction) | Not represented |
| Key length encoding | In fixed header (`ksz`) | Separate 4-byte prefix after fixed header | In fixed header (`key_size`) |

The `hash-index-storage` variant is closer to the paper's spirit (hint files are self-sufficient for keydir reconstruction) but trades away integrity checking. The `log-structured-hash-table` variant prioritizes data-path correctness (CRC on every read) but treats hint files as a lossy optimization — on recovery, if hints are missing or corrupt, it falls back to a full segment scan.

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — Where hint files are actually produced; shows how compaction writes merged data files alongside hint files and handles file rotation mid-compaction
- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — The fallback path when no hint file exists; compare its CRC validation logic against the hint-loading path to see the integrity gap
- [function] `hash-index-storage/bitcask.py:_rebuild_index` — The startup decision point: hint file vs. full data scan, and what happens when hint files are partially present
- [general] `bitcask-tombstone-lifecycle` — How tombstones flow through write, compaction, and hint generation in both implementations; the `hash-index-storage` variant filters them during compaction (line 221) while `log-structured-hash-table` uses a sentinel value
- [file] `write-ahead-log/wal.py` — A contrasting approach to CRC usage in the same repo; every WAL record gets CRC32 validation, showing the project does use integrity checks where crash safety matters most

## Beliefs

- `hash-index-hint-self-sufficient` — The `hash-index-storage` hint file contains all four keydir fields (file_id, offset, size, timestamp), making it sufficient to fully reconstruct the in-memory index without reading data files
- `log-structured-hint-incomplete` — The `log-structured-hash-table` hint file stores only (key_size, offset), so the recovered index lacks timestamp and record size compared to the paper's design
- `no-hint-integrity-checking` — Neither Bitcask implementation validates hint file integrity with CRC or checksums; a corrupted hint file silently produces wrong index entries
- `hint-files-compaction-only` — Hint files are only written during compaction (`_write_hint_file` is called from `compact`), never during normal append operations on the active file
- `log-structured-data-crc-hint-gap` — The `log-structured-hash-table` validates CRC32 on every data record read but performs zero validation when loading hint files, creating an integrity asymmetry

