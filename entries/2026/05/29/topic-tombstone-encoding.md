# Topic: How deletes use `TOMBSTONE = b"__BITCASK_TOMBSTONE__"` as a sentinel value within the same record format, and the implications for key-size/value-size calculations

**Date:** 2026-05-29
**Time:** 12:29

## How Deletes Work via Tombstone Sentinel Values

The two Bitcask implementations in this repo take **fundamentally different approaches** to representing deletes, and the difference has real implications for correctness.

### The Sentinel Approach (`log-structured-hash-table/bitcask.py`)

This implementation defines an explicit sentinel constant at line 9:

```python
TOMBSTONE = b"__BITCASK_TOMBSTONE__"
```

When `delete()` is called (around line 193), it writes a **normal record** where the value bytes happen to be the sentinel:

```python
def delete(self, key: str) -> bool:
    ...
    self._write_record(key, TOMBSTONE)
    self._index.pop(key, None)
    return existed
```

The critical thing to understand is what `_write_record` (line 152) does with this:

```python
header = struct.pack(HEADER_FMT, crc, len(key_bytes), len(value))
```

The `value_size` field in the header is set to `len(TOMBSTONE)` = **20 bytes**. The record on disk is structurally identical to a `put("mykey", b"__BITCASK_TOMBSTONE__")`. There is no flag, no special header bit — the tombstone is indistinguishable from a real value at the binary format level.

During recovery in `_scan_segment` (around line 93), tombstone detection happens by **comparing the value bytes**:

```python
value = payload[key_size:]
if value == TOMBSTONE:
    self._index.pop(key, None)
```

### The Empty-String Approach (`hash-index-storage/bitcask.py`)

This implementation takes a simpler path. `delete()` (around line 189) writes an empty value:

```python
def delete(self, key: str) -> None:
    self._write_record(key, "")
    self.keydir.pop(key, None)
```

Here, `value_size` in the header is **0**. During recovery in `_scan_data_file` (around line 123), tombstone detection checks the size field directly:

```python
if val_size == 0:
    self.keydir.pop(key, None)
```

And `get()` has a corresponding guard:

```python
if value == "":
    return None
```

### Implications for Key-Size / Value-Size Calculations

| Aspect | `log-structured-hash-table` | `hash-index-storage` |
|---|---|---|
| `value_size` in header | 20 (sentinel length) | 0 |
| Total record size | `HEADER_SIZE + key_len + 20` | `HEADER_SIZE + key_len + 0` |
| Detection method | Byte comparison against sentinel | Integer comparison (`val_size == 0`) |
| CRC coverage | Covers key + tombstone bytes | Covers key only (no value bytes) |
| Forbidden values | `b"__BITCASK_TOMBSTONE__"` cannot be stored | Empty string `""` cannot be stored |

Three concrete consequences:

1. **Disk overhead**: Every tombstone in `log-structured-hash-table` costs 20 extra bytes on disk. In a workload with heavy deletes, this adds up before compaction runs.

2. **Collision risk**: If a caller does `store.put("x", b"__BITCASK_TOMBSTONE__")`, the next recovery will treat that key as deleted. The empty-string approach has an analogous but smaller risk — empty values are arguably a more common legitimate value than a 20-byte magic string.

3. **Hint file blind spot**: `_load_hint_file` in `log-structured-hash-table` (line 107) only stores `(key, offset)` pairs — it does not record whether a record is a tombstone. So hint files cannot represent deletes; they'll re-add tombstoned keys to the index. The `hash-index-storage` version has the same limitation.

---

## Topics to Explore

- [function] `log-structured-hash-table/bitcask.py:compact` — How compaction filters out tombstones and whether the sentinel comparison is consistent with `_scan_segment`
- [function] `log-structured-hash-table/bitcask.py:_load_hint_file` — Whether the hint file format can represent tombstones, or if this is a correctness gap during recovery
- [function] `hash-index-storage/bitcask.py:compact` — Compare how the empty-value approach handles tombstone elimination during merge
- [general] `tombstone-in-lsm-tree` — How the LSM-tree implementation handles deletes versus these Bitcask variants, since LSM trees face the same tombstone-propagation problem across levels
- [general] `crc-coverage-of-tombstones` — Whether the CRC protecting tombstone records actually matters, since a corrupted tombstone that fails CRC could resurrect a deleted key

---

## Beliefs

- `lsht-tombstone-is-real-value` — In `log-structured-hash-table/bitcask.py`, tombstone records have `value_size=20` in the header and are structurally identical to normal records; detection requires reading and comparing the full value bytes
- `hash-index-tombstone-is-zero-length` — In `hash-index-storage/bitcask.py`, tombstone records have `value_size=0` and are detected by checking the size field alone, without reading value bytes
- `hint-files-cannot-represent-deletes` — Neither implementation's hint file format stores tombstone status, so recovery via hint files alone will incorrectly restore deleted keys to the index
- `tombstone-sentinel-is-a-forbidden-value` — Storing `b"__BITCASK_TOMBSTONE__"` as a legitimate value in `log-structured-hash-table` will cause silent data loss on next recovery, as `_scan_segment` will interpret it as a delete

