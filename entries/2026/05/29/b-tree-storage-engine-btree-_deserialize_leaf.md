# Function: _deserialize_leaf in b-tree-storage-engine/btree.py

**Date:** 2026-05-29
**Time:** 07:45

## `_deserialize_leaf` — Leaf Page Deserializer

### Purpose

`_deserialize_leaf` parses a raw byte buffer (a fixed-size disk page) back into the logical components of a B-tree leaf node: the sorted keys, their associated values, and the page number of the next sibling leaf. It's the inverse of `_serialize_leaf` and is the only way the B-tree reads leaf data after loading a page from disk.

### Contract

**Preconditions:**
- `data` is a `bytes` object at least `HEADER_SIZE + 4` bytes long (3-byte header + 4-byte sibling pointer).
- The first byte of `data` encodes page type `LEAF` (1). The function reads but discards this byte — it trusts the caller to have already verified the page type via `_page_type()`.
- The `num_keys` field in the header accurately reflects how many key-value entries follow.
- Keys are UTF-8 encoded. Values are raw bytes.

**Postconditions:**
- Returns a 3-tuple `(keys, values, next_sibling)` where `len(keys) == len(values) == num_keys`.
- `keys` are decoded Python `str` objects; `values` remain as `bytes`.
- `next_sibling` is an unsigned 32-bit page number, or `0xFFFFFFFF` (`NO_SIBLING`) if this is the rightmost leaf.

**Invariant:** The returned keys are in sorted order — but only because `_serialize_leaf` and the insert logic maintain that invariant. This function does not verify sort order.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `data` | `bytes` | A full page buffer, typically `page_size` bytes (default 4096), read via `PageManager.read_page()` |

No edge-case handling for truncated or corrupt data — if the buffer is too short, `struct.unpack` will raise `struct.error`.

### Return Value

```python
(keys: list[str], values: list[bytes], next_sibling: int)
```

- **`keys`** — Sorted list of UTF-8 string keys.
- **`values`** — Corresponding raw byte values, positionally aligned with `keys`.
- **`next_sibling`** — Page number of the next leaf to the right, forming a singly-linked list across all leaves. `NO_SIBLING` (0xFFFFFFFF) marks the end.

The caller must handle the empty case (`keys == []`) — this occurs for a freshly created or fully-deleted leaf.

### Algorithm

```
1. Unpack the 3-byte page header (HEADER_FMT = '>BH'):
   - Byte 0: page type (discarded into `_`)
   - Bytes 1-2: num_keys (big-endian unsigned short)

2. Read the next 4 bytes as the next-sibling pointer (big-endian unsigned int).

3. For each of the `num_keys` entries, sequentially:
   a. Read 2 bytes → key length (klen)
   b. Read klen bytes → key (decode as UTF-8)
   c. Read 2 bytes → value length (vlen)
   d. Read vlen bytes → value (kept as raw bytes)
   e. Append to keys[] and values[]

4. Return (keys, values, next_sib)
```

The wire format per entry is: `[klen:2B][key:klenB][vlen:2B][value:vlenB]` — a classic length-prefixed TLV without the type tag. Maximum key or value size is 65,535 bytes (2^16 - 1), though in practice the page size (default 4096) constrains this much further.

### Side Effects

None. Pure function — no I/O, no mutation, no state changes. It operates entirely on the byte buffer passed in.

### Error Handling

No explicit error handling. The function will raise:

- **`struct.error`** — if `data` is too short for any `struct.unpack` call (truncated page).
- **`UnicodeDecodeError`** — if a key contains invalid UTF-8 bytes.
- **`IndexError`** — if `data` is shorter than the offsets implied by the length prefixes (corrupt length fields).

All of these propagate uncaught. The function assumes well-formed pages — reasonable given the WAL provides crash safety and only `_serialize_leaf` produces the data.

### Usage Patterns

Every B-tree operation that touches leaf data calls this function:

- **`_search`** (via `get`) — find a key by binary search on the deserialized keys
- **`_insert`** (via `put`) — deserialize, insert into the lists, re-serialize
- **`_delete`** — deserialize, remove from the lists, re-serialize
- **`range_scan`** — walk the sibling chain, deserializing each leaf
- **`__iter__`** — same leaf-chain walk
- **`min_key` / `max_key`** — deserialize to read the first/last key

The typical call pattern is:

```python
data = self.pm.read_page(page_num)
keys, values, next_sib = _deserialize_leaf(data)
```

Callers always pair this with `PageManager.read_page()`. After mutation, they re-serialize with `_serialize_leaf` and write back through the WAL.

### Dependencies

- **`struct`** (stdlib) — all binary unpacking
- **Module-level constants** — `HEADER_FMT` (`'>BH'`), `HEADER_SIZE` (3 bytes). These are shared with `_serialize_leaf`, `_deserialize_internal`, and `_serialize_internal`, forming the common page-header contract.

No external dependencies.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_serialize_leaf` — The inverse operation; understanding the pair together reveals the exact wire format and the symmetry contract
- [function] `b-tree-storage-engine/btree.py:_deserialize_internal` — Same pattern but for internal nodes; comparing the two shows why leaves store a sibling pointer while internals store child pointers
- [function] `b-tree-storage-engine/btree.py:BTree.range_scan` — The primary consumer of the `next_sibling` pointer; demonstrates how the leaf chain enables efficient sequential access without traversing internal nodes
- [general] `page-overflow-and-split-mechanics` — How `_insert` decides when a leaf is full and splits it, and how the serialized size interacts with `max_keys` and `page_size`
- [file] `b-tree-storage-engine/test_btree.py` — Test cases that exercise round-trip serialization and edge cases like empty leaves and maximum-size entries

---

## Beliefs

- `leaf-wire-format-is-header-sibling-then-kv-pairs` — Leaf page binary layout is: 3-byte header (type + num_keys), 4-byte next-sibling pointer, then N length-prefixed key-value pairs, each with 2-byte key length, key bytes, 2-byte value length, value bytes.
- `leaf-keys-are-utf8-values-are-raw-bytes` — Deserialization decodes keys as UTF-8 strings but leaves values as raw `bytes`; this asymmetry means keys must be valid UTF-8 while values are opaque.
- `max-key-or-value-size-is-65535-bytes` — Key and value lengths are stored as `>H` (unsigned 16-bit), capping individual key or value size at 65,535 bytes, though page size is the practical limit.
- `deserialize-leaf-is-pure-and-trusts-input` — The function performs no validation of page type, sort order, or buffer length; it assumes the WAL and serialize functions guarantee well-formed pages.
- `sibling-pointer-enables-sequential-scan-without-parent-traversal` — The `next_sibling` field in each leaf forms a singly-linked list across all leaves, allowing `range_scan` and `__iter__` to walk leaves left-to-right without re-descending from the root.

