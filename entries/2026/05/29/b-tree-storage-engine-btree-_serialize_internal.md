# Function: _serialize_internal in b-tree-storage-engine/btree.py

**Date:** 2026-05-29
**Time:** 07:46

## `_serialize_internal` — Serialize a B-Tree Internal Node to Binary

### Purpose

`_serialize_internal` converts an in-memory representation of a B-tree internal (non-leaf) node into a compact binary byte sequence suitable for writing to a fixed-size page on disk. Internal nodes don't store values — they store keys that act as separators and pointers (page numbers) to child pages. This function is the counterpart to `_deserialize_internal`, and together they form the serialization boundary between the in-memory tree operations and the page-based storage layer.

### Contract

**Preconditions:**
- `len(children) == len(keys) + 1` — the B-tree invariant that an internal node with *n* keys has *n+1* children. This is **not checked** by the function; violating it silently produces a corrupt page (the loop indexes `children[i + 1]`, so too few children causes an `IndexError`, but too many silently drops the last child).
- Each key must be either `str` or `bytes`. No other types are handled.
- Each child must be an integer that fits in 4 bytes unsigned (`0` to `2^32 - 1`).
- The serialized output must fit within the page size. This is **not enforced here** — the caller (`BTree._insert`) relies on key-count limits to prevent overflow.

**Postconditions:**
- Returns a `bytes` object encoding the complete internal node in big-endian format.
- The output is a valid input to `_deserialize_internal` (round-trip identity).

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `keys` | `list[str \| bytes]` | Separator keys in sorted order. Determines which child subtree a search descends into. |
| `children` | `list[int]` | Page numbers of child nodes. `children[i]` contains all keys < `keys[i]`; `children[i+1]` contains all keys >= `keys[i]`. |

**Edge cases:**
- Empty `keys` (single-child internal node): only the header and `children[0]` are written. The loop body never executes. This is a degenerate but valid state that can arise transiently during splits.

### Return Value

`bytes` — the serialized page data, **not yet padded** to page size. The caller (`_wal_write_page` → `PageManager.write_page`) pads with null bytes via `.ljust(page_size, b'\x00')`.

### Algorithm

```
1. Pack a 3-byte header: page type (INTERNAL = 0) as uint8, key count as uint16 big-endian.
2. Pack children[0] as a 4-byte big-endian uint32 — the leftmost child pointer.
3. For each key at index i:
   a. Encode the key to UTF-8 bytes if it's a str (no-op if already bytes).
   b. Pack a 2-byte key length prefix (uint16), then the raw key bytes.
   c. Pack children[i+1] as a 4-byte uint32 — the child pointer to the right of this key.
4. Return the accumulated buffer.
```

The binary layout is:

```
[type:1B][num_keys:2B][child_0:4B] ([key_len:2B][key_data:varB][child_i+1:4B])...
```

This interleaved layout mirrors the logical structure: start with the leftmost child, then alternate (separator key, right child). It's the standard on-disk format for B-tree internal nodes described in DDIA Chapter 3.

### Side Effects

None. This is a pure function — no I/O, no mutations, no state changes.

### Error Handling

- **`IndexError`**: raised if `len(children) < len(keys) + 1` (too few children). Not caught.
- **`struct.error`**: raised if a child page number exceeds `2^32 - 1` or key length exceeds `2^16 - 1` (65535 bytes). Not caught.
- **`AttributeError`**: raised if a key is neither `str` nor `bytes` (no `.encode` method). Not caught.
- Extra children beyond `len(keys) + 1` are **silently dropped** — a subtle data-loss bug if the invariant is violated.

No exceptions are intentionally raised or caught. The function trusts its callers to uphold the B-tree structural invariant.

### Usage Patterns

Called in three places within `BTree`:

1. **`put` → root split** (`btree.py:213`): creates a new root internal node with one key and two children (the old root and the new page from the split).
2. **`_insert` → internal node update after child split** (`btree.py:249`): re-serializes an internal node after inserting a promoted key and new child pointer.
3. **`_delete` → internal node update after child removal** (`btree.py:286`): re-serializes after removing a key and child pointer when an empty leaf is cleaned up.

The caller obligation is to ensure the key/children invariant holds and that the resulting byte sequence doesn't exceed `page_size`.

### Dependencies

| Dependency | Usage |
|------------|-------|
| `struct` (stdlib) | Binary packing via `struct.pack` with big-endian format strings |
| `HEADER_FMT` (`'>BH'`) | 3-byte header format: 1 byte type + 2 byte key count |
| `INTERNAL` (`0`) | Page type constant written into the header |

No external dependencies. The function uses only Python stdlib and module-level constants.

### Design Notes

The key length is stored as `uint16` (`'>H'`), capping individual keys at 65535 bytes. The key count in the header is also `uint16`, capping an internal node at 65535 keys — far beyond what would fit in a typical 4KB page. The actual limit is enforced by `self.max_keys` in `BTree._insert`.

The function does not include any checksum or page-level integrity check — that responsibility falls to the WAL layer, which wraps each page write with a CRC32.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_deserialize_internal` — The inverse operation; verify the round-trip contract and see how the binary layout is unpacked
- [function] `b-tree-storage-engine/btree.py:_serialize_leaf` — Compare leaf serialization (which adds values and a sibling pointer) to understand why internal and leaf nodes have different formats
- [function] `b-tree-storage-engine/btree.py:BTree._insert` — The primary caller; shows how splits produce the keys/children arrays and how overflow triggers the split path
- [general] `b-tree-page-overflow` — What happens when serialized data exceeds page_size? The max_keys heuristic uses a conservative estimate but doesn't account for variable-length keys
- [file] `b-tree-storage-engine/test_btree.py` — Check whether tests exercise the edge cases: empty keys list, maximum key length, round-trip serialization fidelity

## Beliefs

- `internal-node-children-invariant` — `_serialize_internal` assumes `len(children) == len(keys) + 1` but does not assert it; violation causes silent data loss (extra children dropped) or IndexError (too few children)
- `internal-page-binary-layout` — Internal pages are laid out as `[type:1B][num_keys:2B][child0:4B]([keylen:2B][key:varB][child:4B])...`, all big-endian, with no alignment padding
- `serialize-internal-is-pure` — `_serialize_internal` is a pure function with no side effects, I/O, or state mutation
- `key-length-cap-uint16` — Individual key sizes are capped at 65535 bytes by the `uint16` length prefix; exceeding this raises `struct.error` at pack time rather than being checked upfront
- `no-page-size-enforcement-in-serialize` — Neither `_serialize_internal` nor `_serialize_leaf` checks whether the output fits within `page_size`; overflow prevention is the caller's responsibility via `max_keys`

