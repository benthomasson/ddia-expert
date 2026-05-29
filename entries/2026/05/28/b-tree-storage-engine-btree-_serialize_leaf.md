# Function: _serialize_leaf in b-tree-storage-engine/btree.py

**Date:** 2026-05-28
**Time:** 19:15

## `_serialize_leaf` — Leaf Page Serializer

### Purpose

`_serialize_leaf` converts an in-memory representation of a B-tree leaf node (lists of keys and values) into a compact binary format suitable for writing to a fixed-size disk page. It's the write half of the leaf serialization pair — `_deserialize_leaf` is its inverse.

This exists because the B-tree operates on in-memory lists during mutations (insert, delete, update) but persists data as raw bytes in a page-oriented file. Every time a leaf changes, the engine rebuilds the binary page from scratch rather than doing in-place byte surgery.

### Contract

**Preconditions:**
- `keys` and `values` are equal-length sequences
- Each key is either `str` or `bytes`
- Each value is `bytes` (no string coercion is applied to values)
- The serialized output must fit within the page size — **this is NOT enforced here**; the caller is responsible (see `BTree.put` which checks `entry_size` before insertion)
- Key lengths and value lengths must each fit in an unsigned 16-bit integer (< 65,536 bytes)

**Postconditions:**
- Returns a `bytes` object with the layout: `[header(3B)][next_sibling(4B)][entries...]`
- The output is a valid input to `_deserialize_leaf` (round-trip identity)

**Invariant:** The order of entries in the output matches the order of `keys`/`values` — the function does not sort.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `keys` | `list[str \| bytes]` | Sorted key sequence. Strings are UTF-8 encoded; bytes pass through as-is. |
| `values` | `list[bytes]` | Corresponding values. Must be raw bytes — no encoding is applied. |
| `next_sibling` | `int` | Page number of the next leaf in the linked list. Defaults to `0xFFFFFFFF` (sentinel meaning "no next leaf"). |

**Edge cases:** Empty `keys`/`values` is valid — produces a 7-byte buffer (header + sibling pointer) with zero entries. This is used during tree initialization (`_write_empty_leaf`).

### Return Value

A `bytes` object — the raw page content (unpadded). The caller (`_wal_write_page` → `PageManager.write_page`) pads it to `page_size` with null bytes before writing to disk.

### Algorithm

1. **Pack the header** (3 bytes): page type `LEAF` (1 byte, value `0x01`) + number of keys as big-endian unsigned 16-bit (`>BH`).
2. **Pack the sibling pointer** (4 bytes): big-endian unsigned 32-bit (`>I`).
3. **For each key-value pair:**
   - Encode the key to UTF-8 if it's a `str`; use as-is if already `bytes`.
   - Write key length as big-endian unsigned 16-bit (`>H`), then the key bytes.
   - Write value length as big-endian unsigned 16-bit (`>H`), then the value bytes.
4. **Return** the accumulated buffer.

The wire format per entry is: `[key_len: 2B][key: variable][val_len: 2B][val: variable]` — a standard length-prefixed TLV layout without the type.

### Side Effects

None. Pure function — no I/O, no mutation, no state changes.

### Error Handling

No explicit error handling. Will raise implicitly if:
- `struct.pack('>H', len(kb))` overflows — key longer than 65,535 bytes raises `struct.error`
- `v` is not a bytes-like object — concatenation with `buf` raises `TypeError`
- `keys` and `values` have different lengths — `zip` silently truncates to the shorter, producing a corrupt page (silent data loss)

### Usage Patterns

Called in three contexts:

1. **Leaf insert/update** (`_insert`, depth==1): after modifying the in-memory key/value lists, reserializes the whole leaf.
2. **Leaf split** (`_insert`): serializes both left and right halves after splitting, with the right page inheriting the old `next_sibling` and the left page pointing to the new right page.
3. **Leaf delete** (`_delete`): reserializes after removing an entry; also used when relinking sibling pointers during empty-leaf cleanup.

Callers always pass the result to `_wal_write_page`, never directly to `PageManager`.

### Dependencies

- `struct` — binary packing (standard library)
- Module-level constants: `HEADER_FMT` (`'>BH'`), `LEAF` (`1`), `NO_SIBLING` (`0xFFFFFFFF`)

No external dependencies.

### Assumptions Not Enforced

1. **`len(keys) == len(values)`** — `zip` silently drops extras, corrupting the `num_keys` count in the header.
2. **Total serialized size ≤ page_size** — overflow is truncated silently by `PageManager.write_page`, corrupting the page.
3. **Values are `bytes`** — no type check or encoding; passing a `str` value crashes at concatenation.
4. **Key/value length < 2^16** — `struct.pack('>H', ...)` will raise, but no friendly error message.
5. **Keys are already sorted** — the function preserves order but doesn't verify it; an unsorted input produces a valid-looking but logically broken leaf (binary search in `_search` would fail).

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_deserialize_leaf` — The inverse operation; understanding the round-trip is essential to verifying correctness of the wire format
- [function] `b-tree-storage-engine/btree.py:_serialize_internal` — Compare with leaf serialization: internal nodes store child pointers instead of values and have no sibling link
- [function] `b-tree-storage-engine/btree.py:BTree._insert` — The primary caller; shows how splits produce two `_serialize_leaf` calls with carefully threaded sibling pointers
- [general] `page-overflow-and-size-limits` — The page size is fixed but serialization has no overflow guard; trace how `BTree.put` and `max_keys` cooperate to prevent corruption
- [file] `b-tree-storage-engine/test_btree.py` — Test coverage for edge cases like empty trees, single-entry leaves, and split behavior

---

## Beliefs

- `leaf-wire-format-is-header-sibling-entries` — Leaf pages are laid out as `[type:1B][num_keys:2B][next_sibling:4B][entries...]` where each entry is `[key_len:2B][key][val_len:2B][val]`, all big-endian.
- `serialize-leaf-is-pure` — `_serialize_leaf` is a pure function with no side effects; all I/O happens in the caller via `_wal_write_page`.
- `leaf-sibling-forms-linked-list` — Leaf nodes form a singly-linked list via `next_sibling`, enabling sequential range scans without traversing internal nodes.
- `zip-truncation-risk` — If `keys` and `values` have mismatched lengths, `zip` silently truncates and the `num_keys` header field will be wrong, producing a corrupt page.
- `values-must-be-bytes` — Values are written raw with no encoding; keys accept `str` (auto-encoded to UTF-8) or `bytes`, but values must already be `bytes`.

