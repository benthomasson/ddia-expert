# Topic: The empty-string tombstone in `hash-index-storage` means you literally cannot store an empty value; compare with the dedicated sentinel in `log-structured-hash-table` and consider what the paper recommends (a flag in the record header)

**Date:** 2026-05-29
**Time:** 08:54

## The Tombstone Design Flaw in `hash-index-storage`

### How `hash-index-storage` encodes deletion

In `hash-index-storage/bitcask.py`, deletion works by writing a record with an empty string as the value:

```python
# bitcask.py:181
def delete(self, key: str) -> None:
    """Delete a key by appending a tombstone record."""
    self._maybe_rotate()
    self._write_record(key, "")
    self.keydir.pop(key, None)
```

The `get` method then treats any empty value as "deleted":

```python
# bitcask.py:173-177
def get(self, key: str) -> Optional[str]:
    ...
    if value == "":
        return None
    return value
```

And during index rebuild, the same convention is used — `val_size == 0` means tombstone:

```python
# bitcask.py:137-139
if val_size == 0:
    # Tombstone - remove from keydir
    self.keydir.pop(key, None)
```

This creates a **semantic collision**: the empty string `""` serves double duty as both "this key was deleted" and "the value is an empty string." A caller who does `store.put("config", "")` followed by `store.get("config")` gets `None` back — the store silently treats their valid write as a deletion. There's no way to distinguish intent.

### How `log-structured-hash-table` avoids this

The second implementation (`log-structured-hash-table/bitcask.py`) uses a dedicated byte sentinel:

```python
# bitcask.py:9
TOMBSTONE = b"__BITCASK_TOMBSTONE__"
```

Deletion writes this specific byte sequence as the value:

```python
# bitcask.py:188-194
def delete(self, key: str) -> bool:
    """Delete a key by writing a tombstone. Returns True if key existed."""
    ...
    self._write_record(key, TOMBSTONE)
    self._index.pop(key, None)
    return existed
```

And during recovery, only this exact sentinel is treated as a tombstone:

```python
# bitcask.py:102-105
if value == TOMBSTONE:
    self._index.pop(key, None)
else:
    self._index[key] = (seg_path, offset)
```

This is better — empty values work fine. But it shifts the problem rather than eliminating it: now a user who stores the literal bytes `b"__BITCASK_TOMBSTONE__"` as a value will have that key silently deleted on recovery. It's unlikely, but the failure mode is silent data loss, which is the worst kind.

### What the paper recommends

DDIA (Chapter 3) describes the Bitcask approach and notes that deletions should use a **special deletion record** — a tombstone. The clean way to implement this is a **flag in the record header**. Instead of overloading the value field, you add a single byte (or bit) to the header format that says "this record is a delete." The header already carries metadata (`timestamp`, `key_size`, `value_size`); a `flags` field fits naturally:

```
HEADER_FORMAT = "<dIIB"  # timestamp, key_size, value_size, flags
# flags & 0x01 → tombstone
```

With a header flag:
- Empty string is a valid, storable value
- No magic sentinel can collide with user data
- The tombstone status is unambiguous at the structural level
- Compaction can skip tombstones without even reading the value bytes

The `hash-index-storage` header format (`<dII` — timestamp, key_size, value_size, 16 bytes total at line 10) would only need one additional byte to support this properly.

### Summary of the three approaches

| Approach | Implementation | Flaw |
|----------|---------------|------|
| Empty string as tombstone | `hash-index-storage/bitcask.py` | Cannot store `""` as a value |
| Magic byte sentinel | `log-structured-hash-table/bitcask.py` | Cannot store the sentinel bytes as a value |
| Header flag | (DDIA recommendation) | No value-space collision; cleanest separation of data from metadata |

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — See whether compaction correctly skips tombstone records or if the empty-string ambiguity causes data loss during merge
- [function] `log-structured-hash-table/bitcask.py:compact` — Compare how compaction handles the dedicated sentinel vs the empty-string approach
- [function] `hash-index-storage/bitcask.py:_write_record` — The header format `<dII` has no flags field; trace how a header-flag fix would ripple through write, read, and rebuild
- [general] `crc-integrity-on-tombstones` — `log-structured-hash-table` CRC-checks every record including tombstones; `hash-index-storage` has no CRC at all — consider what happens when a corrupted record has `val_size == 0` by accident
- [general] `hint-file-tombstone-handling` — Hint files in `hash-index-storage` (line 150–160) don't encode tombstone status, so a deleted key with a hint entry would resurrect on restart

## Beliefs

- `hash-index-empty-string-is-tombstone` — In `hash-index-storage/bitcask.py`, `put(key, "")` is indistinguishable from `delete(key)`: both produce a zero-length value record, and `get` returns `None` for both
- `log-structured-sentinel-is-in-band` — `log-structured-hash-table/bitcask.py` uses `TOMBSTONE = b"__BITCASK_TOMBSTONE__"` as an in-band sentinel; storing those exact bytes as a value causes silent data loss on recovery
- `hash-index-no-crc` — `hash-index-storage/bitcask.py` has no CRC or checksum field in its record header (`<dII`), so a bit-flip that zeroes `value_size` silently converts a live record into a tombstone
- `header-flag-zero-value-collision` — Neither implementation uses a header flag for tombstones; both encode deletion status within the value payload, violating the principle that metadata belongs in the record header

