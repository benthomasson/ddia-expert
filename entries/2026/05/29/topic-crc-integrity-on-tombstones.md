# Topic: `log-structured-hash-table` CRC-checks every record including tombstones; `hash-index-storage` has no CRC at all — consider what happens when a corrupted record has `val_size == 0` by accident

**Date:** 2026-05-29
**Time:** 12:59

## CRC as a Corruption Firewall: Why `val_size == 0` Is Catastrophic Without It

The two modules implement the same Bitcask architecture but differ fundamentally in how they handle data integrity — and that difference turns a single corrupted byte into either a clean error or a silent data-loss cascade.

### How each module detects tombstones

**`log-structured-hash-table/bitcask.py`** uses an explicit sentinel value:

```python
TOMBSTONE = b"__BITCASK_TOMBSTONE__"          # line 9
```

A delete writes a full record where the value *is* that 20-byte sentinel (`delete` at line ~179 calls `_write_record(key, TOMBSTONE)`). The tombstone has a non-zero `value_size` of 20 and is CRC-protected like any other record. During `_scan_segment` (line ~93), the scanner checks `if value == TOMBSTONE` *after* the CRC passes.

**`hash-index-storage/bitcask.py`** uses an empty value as the tombstone:

```python
def delete(self, key: str) -> None:
    self._write_record(key, "")               # line ~197
```

During `_scan_data_file` (line ~122), the scanner checks `if val_size == 0` in the header to detect tombstones (line ~137). There is no CRC — no `zlib` import, no checksum field in the `<dII` header format (line 10).

### The corruption scenario: `val_size` flips to 0

Suppose a record for key `"user:42"` with a 100-byte value suffers a bit-flip that zeros out the `value_size` field in the on-disk header.

**In `log-structured-hash-table`**, `_scan_segment` (line ~93) does this:

1. Reads the header: `crc, key_size, value_size=0`
2. Reads `payload = f.read(key_size + 0)` — only the key bytes
3. Computes `expected_crc = zlib.crc32(payload)` — CRC of just the key
4. Compares against the stored CRC, which was computed over `key + original_100_byte_value`
5. **Mismatch.** The scanner hits `break` and stops — the corrupted record and everything after it are safely abandoned

The CRC catches it because the checksum covers the *payload*, and a `val_size` of 0 means the scanner reads the wrong number of bytes. The math can't agree.

**In `hash-index-storage`**, `_scan_data_file` (line ~122) does this:

1. Reads the header: `ts, key_size, val_size=0`
2. Reads `key_bytes = reader.read(key_size)` — correct key
3. Reads `reader.read(0)` — skips zero bytes (the real 100-byte value is still ahead)
4. Sees `val_size == 0`, enters the tombstone branch: **`self.keydir.pop(key, None)`**
5. The key `"user:42"` is silently deleted from the index
6. The file cursor is now 100 bytes behind where it should be — pointing into the *middle* of the old value
7. The next loop iteration reads those value bytes as a *new record header*, parsing garbage as `timestamp`, `key_size`, and `val_size`

Step 7 is where a single corrupted field cascades into total index corruption. The scanner chases garbage offsets through the rest of the file, potentially:

- Misinterpreting value data as keys (and inserting phantom entries into `keydir`)
- Hitting an accidental `val_size == 0` in the garbage and deleting *more* real keys
- Reaching a decode error on `key_bytes.decode("utf-8")` if the garbage isn't valid UTF-8 — which crashes the entire recovery

### Why the design choice matters

The LSHT approach has two layers of defense:

1. **CRC catches any field corruption** — including `key_size`, `val_size`, or the payload itself
2. **Tombstone is a value, not a size** — so `val_size == 0` has no special semantic meaning; it's just a record with an empty value, still CRC-checked

The HIS approach overloads `val_size == 0` to mean "deleted," which creates an implicit coupling between the wire format and deletion semantics. Any corruption path that produces a zero in that 4-byte field triggers a false tombstone — and without a checksum, there is nothing to distinguish "intentional zero" from "accidental zero."

### The `get` path is also affected

Even outside of recovery, `hash-index-storage/bitcask.py:get` (line ~185) uses the `size` from `KeyEntry` to read the record. If the in-memory index is correct but the on-disk data is corrupted, the only check is an `assert read_key == key` (line ~187). There is no integrity verification of the value — a corrupted value is returned silently to the caller.

In contrast, `log-structured-hash-table/bitcask.py:get` (line ~163) performs a full CRC check on every read and raises `CorruptionError` on mismatch.

---

## Topics to Explore

- [function] `log-structured-hash-table/bitcask.py:compact` — Compaction rewrites records to new segments; check whether CRC is recomputed and whether tombstones survive compaction or are dropped
- [function] `hash-index-storage/bitcask.py:_load_hint_file` — Hint files bypass the data file scanner entirely; since there's no CRC, a corrupted hint file could inject phantom index entries with no way to detect it
- [general] `tombstone-encoding-strategies` — Compare explicit sentinel (`__BITCASK_TOMBSTONE__`) vs. zero-length value vs. a flags byte in the header — each has different corruption surface area
- [function] `hash-index-storage/bitcask.py:compact` — During compaction, corrupted records in immutable files are re-read and rewritten; without CRC, corruption gets *permanently baked* into the compacted output
- [general] `partial-write-recovery` — Both implementations break on partial writes (incomplete headers), but only LSHT can distinguish "partial write" from "corrupted complete write"

## Beliefs

- `lsht-crc-covers-key-and-value` — In `log-structured-hash-table`, the CRC32 is computed over the concatenation of key bytes and value bytes (`payload = key_bytes + value`), meaning corruption in *any* field — key, value, key_size, or value_size — causes a checksum mismatch
- `his-tombstone-is-zero-val-size` — In `hash-index-storage`, deletion is encoded as `val_size == 0` in the record header, creating an implicit equivalence between "empty value" and "deleted" that cannot be distinguished from corruption
- `his-no-integrity-check-on-read` — `hash-index-storage/bitcask.py:get` returns whatever bytes are on disk without any integrity verification beyond a key-match assert, so corrupted values are served silently
- `lsht-scan-stops-on-first-crc-failure` — `log-structured-hash-table/bitcask.py:_scan_segment` breaks out of the loop on the first CRC mismatch, treating the remainder of the segment as unrecoverable rather than attempting to skip past the bad record
- `his-val-size-corruption-cascades` — A corrupted `val_size` in `hash-index-storage` causes the file cursor to desync, making the scanner misparse all subsequent records in that data file

