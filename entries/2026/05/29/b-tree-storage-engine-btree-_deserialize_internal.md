# Function: _deserialize_internal in b-tree-storage-engine/btree.py

**Date:** 2026-05-29
**Time:** 10:35

## `_deserialize_internal` — B-Tree Internal Node Deserializer

### Purpose

`_deserialize_internal` reconstructs an internal (non-leaf) B-tree node from its raw binary page data. Internal nodes store separator keys and child page pointers that guide search traversal down the tree. This function is the inverse of `_serialize_internal` — together they form the codec for the internal node wire format.

### Contract

**Preconditions:**
- `data` is a `bytes` object of at least `HEADER_SIZE` (3 bytes), and contains a validly serialized internal node.
- The page type byte at `data[0]` should be `INTERNAL` (0), though the function **does not check this** — it reads and discards the type field.
- All keys were encoded as UTF-8 when serialized.

**Postconditions:**
- Returns `(keys, children)` where `len(children) == len(keys) + 1` — the fundamental B-tree invariant that an internal node with *n* keys has *n+1* child pointers.
- `keys` preserves insertion order, which must be sorted for binary search to work in callers.
- `children` are unsigned 32-bit page numbers corresponding to the subtrees between and around the keys.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `data` | `bytes` | Raw page bytes, typically 4096 bytes read from disk via `PageManager.read_page()` |

There are no edge-case guards — passing truncated data, a leaf page, or corrupted bytes will produce silent garbage or raise `struct.error`.

### Return Value

A 2-tuple `(keys: list[str], children: list[int])`:

- `keys` — separator keys as decoded UTF-8 strings. For a node with 0 keys, this is `[]`.
- `children` — child page numbers as Python ints. Always has exactly one more element than `keys`. For a node with 0 keys, this is `[child0]`.

The caller is responsible for using `bisect_right` on `keys` to determine which child pointer to follow (see `_search` and `_find_leaf`).

### Algorithm

1. **Read header** (bytes 0–2): Unpack `HEADER_FMT` (`'>BH'` = 1-byte type + 2-byte key count). The type byte is discarded via `_`.

2. **Read child₀** (bytes 3–6): The first child pointer is stored before any keys, because an internal node with *n* keys has *n+1* children. This is `children[0]` — the subtree for all keys less than `keys[0]`.

3. **Loop `num_keys` times**, reading interleaved key-child pairs:
   - **Key length** (2 bytes, big-endian unsigned short) → `klen`
   - **Key bytes** (`klen` bytes) → decoded from UTF-8 to `str`
   - **Child pointer** (4 bytes, big-endian unsigned int) → page number of the child subtree for keys between `keys[i]` and `keys[i+1]`

The binary layout is:

```
[type:1B][num_keys:2B][child₀:4B] [klen:2B][key][child₁:4B] [klen:2B][key][child₂:4B] ...
```

### Side Effects

None. Pure function — no I/O, no mutation, no state changes.

### Error Handling

No explicit error handling. Failure modes:

| Scenario | Result |
|----------|--------|
| Truncated `data` | `struct.error` from short unpack |
| Corrupted key length (too large) | `IndexError` or reading into adjacent data / padding |
| Non-UTF-8 key bytes | `UnicodeDecodeError` from `.decode('utf-8')` |
| Leaf page passed as input | Silently misinterprets leaf data as internal node format — wrong results, no error |

The function trusts that the caller has already verified the page type via `_page_type(data)` or knows the depth.

### Usage Patterns

Every call site first determines the node type by depth or by reading the type byte, then dispatches:

```python
# In _search (btree.py:204)
ikeys, children = _deserialize_internal(data)
idx = bisect_right(ikeys, key)
return self._search(children[idx], key, depth - 1)
```

Callers include `_search`, `_insert`, `_delete`, `_find_leaf`, `_count_pages`, `_print_node`, `min_key`, `max_key`, and `__iter__`. The pattern is always: deserialize → use `bisect_right`/`bisect_left` on keys → index into children → recurse.

### Dependencies

- `struct` — binary packing/unpacking (standard library)
- Module-level constants `HEADER_FMT` (`'>BH'`) and `HEADER_SIZE` (3) — shared with leaf serialization

### Assumptions Not Enforced

1. **Page type is not validated.** The function will happily parse a leaf page as if it were internal, producing nonsense.
2. **Key ordering is assumed.** If keys were serialized out of order, deserialization succeeds but downstream `bisect_right` calls silently return wrong indices.
3. **Data length is not bounds-checked.** The function assumes `data` is long enough for all declared keys — a `num_keys` field that exceeds actual content causes a crash.
4. **No checksum or integrity verification.** Corruption in the page data is not detected at this layer (the WAL checksum covers the whole page at write time, not at read time).

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_serialize_internal` — The inverse operation; understanding both directions clarifies the exact binary layout and round-trip guarantees
- [function] `b-tree-storage-engine/btree.py:_search` — Primary consumer; shows how deserialized keys and children drive tree traversal via `bisect_right`
- [function] `b-tree-storage-engine/btree.py:_insert` — Shows what happens when an internal node splits: keys and children are deserialized, mutated, re-serialized, and potentially split across two pages
- [general] `btree-internal-vs-leaf-layout` — The structural difference between leaf layout (keys + values + next-sibling pointer for range scans) and internal layout (keys + child pointers, no sibling chain) reflects the dual role of B-tree nodes
- [function] `b-tree-storage-engine/btree.py:_deserialize_leaf` — Direct counterpart for leaf nodes; compare the two to see how the sibling pointer and value fields replace the child pointer array

---

## Beliefs

- `internal-node-children-keys-invariant` — `_deserialize_internal` always returns `len(children) == len(keys) + 1`, maintaining the B-tree invariant that n keys partition n+1 subtrees
- `internal-deserialize-no-type-check` — `_deserialize_internal` discards the page type byte without validating it equals `INTERNAL`; callers must ensure they pass internal-node data
- `internal-node-binary-layout` — Internal nodes store child₀ before any keys, then interleave (key-length, key-bytes, child-pointer) pairs, all big-endian
- `deserialize-internal-pure-function` — `_deserialize_internal` is a pure function with no side effects, I/O, or state mutations
- `internal-keys-utf8-assumption` — `_deserialize_internal` unconditionally decodes key bytes as UTF-8, which will raise `UnicodeDecodeError` if the stored bytes are not valid UTF-8

