# Topic: The SSTable uses a magic number for file-type validation but no per-entry checksums; explore whether the magic check gives any meaningful corruption protection beyond "wrong file type"

**Date:** 2026-05-29
**Time:** 12:06

## Magic Number Validation: File-Type Gate, Not Corruption Shield

The SSTable's magic number check lives in `SSTableReader.__init__` (line ~108 of `sstable.py`):

```python
magic, version, self._entry_count = struct.unpack(HEADER_FMT, f.read(HEADER_SIZE))
assert magic == MAGIC, f"Invalid magic: {magic}"
```

The magic is the 4-byte literal `b"SSTB"` (line 10), written once at file offset 0 by `SSTableWriter.__init__` (line 52). That's the **only integrity check in the entire read path**. Let's be precise about what it catches and what it doesn't.

### What the magic check actually protects against

1. **Wrong file type** — opening a JPEG, a log file, or another binary format. This is the intended purpose.
2. **Completely zeroed file** — `\x00\x00\x00\x00` != `SSTB`.
3. **Gross header corruption** — if the first 4 bytes are mangled, you get a clear error instead of silently misinterpreting the version and entry count.

### What it does NOT protect against

**Everything else.** Specifically:

- **Bit-flips in entry data** — `_read_entry` (lines ~140–163) trusts every byte it reads. A corrupted `key_len` field (`struct.unpack(">H", data)`) produces a wrong-length read. If the corrupted length is smaller than actual, you silently get a truncated key and then misparse everything after it. If larger, you read into the next entry's bytes and decode garbage as UTF-8 (or crash on a decode error, which is accidental detection, not intentional).

- **Corrupted timestamps** — the 8-byte double (`struct.unpack(">d", ts_data)`) will happily decode any 8 bytes into a float. A bit-flip produces a silently wrong timestamp, which can cause **wrong merge resolution** during compaction since `merge_sstables` picks the entry with the highest timestamp.

- **Tombstone/value ambiguity** — the tombstone marker is `0xFF` (line 12). A single-byte corruption that turns a value-length prefix's first byte into `0xFF` converts a live entry into a tombstone. The data is silently treated as deleted. Conversely, a corrupted tombstone marker could cause the reader to interpret subsequent entry bytes as a value length.

- **Footer corruption** — the footer at the file's end stores `index_offset` and `index_count`. Corruption here causes the reader to seek to a wrong offset for the sparse index, producing either garbage index entries or a crash. The magic check at byte 0 is irrelevant to this.

- **Truncated files** — if the file is truncated mid-entry, `_read_entry` returns `None` when reads come up short (lines 143, 148, 152). This silently drops the remaining entries rather than signaling corruption. The entry count in the header is never cross-checked against actual entries read.

### The header entry count is also unchecked

`SSTableWriter.finish()` (lines 88–95) seeks back to byte 0 and rewrites the header with the final count. But `SSTableReader` reads this count and stores it as `self._entry_count` — it never verifies that scanning the data section actually yields that many entries. A corrupted count leads to wrong `metadata()` results and potentially wrong compaction decisions.

### Bottom line

The magic check is a **file-type gate**: it prevents opening non-SSTable files. It provides essentially **zero corruption protection** for the data that matters — keys, values, timestamps, and structural offsets. Any production SSTable implementation would add per-block or per-entry CRC32 checksums (as LevelDB/RocksDB do), plus a footer checksum covering the index. The test suite (`test_sstable.py`) has no corruption or truncation tests, which confirms this wasn't a design goal.

## Topics to Explore

- [general] `per-block-checksums-leveldb` — How LevelDB's block format uses CRC32c on every block to catch exactly the corruption classes this implementation misses
- [function] `sstable-and-compaction/sstable.py:_read_entry` — Trace how a corrupted `key_len` cascades into misaligned reads for all subsequent entries
- [function] `sstable-and-compaction/sstable.py:merge_sstables` — How a corrupted timestamp could silently pick the wrong entry during compaction resolution
- [general] `tombstone-value-ambiguity` — The 0xFF marker is a single-byte sentinel; explore how a one-bit flip could toggle between "deleted" and "live" semantics
- [file] `sstable-and-compaction/test_sstable.py` — No corruption tests exist; designing a test that flips a byte mid-file would demonstrate the silent-corruption problem concretely

## Beliefs

- `sstable-magic-is-only-integrity-check` — The 4-byte magic `b"SSTB"` at offset 0 is the only data integrity check in the entire SSTable read path; no checksums, CRCs, or entry-count cross-validation exist
- `corrupted-key-length-cascades-silently` — A bit-flip in a 2-byte key-length field in `_read_entry` causes all subsequent entries in the file to be misparsed, with no detection mechanism
- `tombstone-marker-single-byte-no-redundancy` — The tombstone/value distinction rests on a single byte (`0xFF` vs first byte of a 4-byte value length), making it vulnerable to single-bit corruption that silently converts live data to deletions or vice versa
- `entry-count-header-never-verified` — The entry count written in the 10-byte header is trusted by `SSTableReader` but never validated against the actual number of entries in the data section

