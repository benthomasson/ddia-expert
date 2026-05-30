# Topic: How `_insert` decides when a leaf is full and splits it, and how the serialized size interacts with `max_keys` and `page_size`

**Date:** 2026-05-29
**Time:** 10:43

I don't have access to read the target repository (`/Users/ben/git/ddia-implementations`). The observations only cover lines 0–200 of a 612-line file, which includes `PageManager`, `WAL`, `_serialize_leaf`, and the beginning of `_deserialize_leaf` — but crucially **not** `_insert` or any split logic. The grep searches in the observations returned zero matches, which suggests they may have been scoped to a different directory.

Here's what I *can* tell you from the available code, and what's missing:

---

## What We Can See: Serialization and Page Constraints

### Leaf serialization format (`_serialize_leaf`, line ~163)

Each leaf page is serialized as:

```
[header: 3B] [next_sibling: 4B] [entries...]
```

Where each entry is:

```
[key_len: 2B] [key: variable] [value_len: 2B] [value: variable]
```

The header (`HEADER_FMT = '>BH'`, line 22) is 3 bytes: 1 byte for page type + 2 bytes for `num_keys`. So the **fixed overhead per leaf page is 7 bytes** (3B header + 4B sibling pointer), and each entry costs `4 + len(key) + len(value)` bytes.

### Page size enforcement (`PageManager.write_page`, line ~79)

`write_page` pads short pages to `page_size` and **silently truncates** pages longer than `page_size` (`data = data[:self.page_size]`). This means if a leaf's serialized content exceeds `page_size`, data will be **silently lost** — there's no overflow mechanism. This makes the split decision critical: it must happen *before* serialization would exceed the page boundary.

### What's Missing

The `_insert` method (somewhere in lines 200–612) is not in the observations. This is where we'd expect to find:

1. **The split trigger** — likely either:
   - A `max_keys` check: `if len(keys) >= max_keys: split()`
   - A size check: `if len(_serialize_leaf(keys, values)) > page_size: split()`
   - Or both (a key count as a proxy, with size as a hard limit)

2. **The split procedure** — how keys are divided between the original and new leaf, how the new page is allocated (`allocate_page`), how the sibling pointer is updated, and how the median key is promoted to the parent.

3. **The `max_keys` parameter** — whether it's derived from `page_size` at construction time or set independently.

Without seeing `_insert`, I cannot confirm how these interact. **To answer this question properly, I need lines ~200–612 of `btree.py`**, specifically the `BTree` class definition and its `_insert` / `_split_leaf` methods.

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:_insert` — The core insertion logic including the split decision; this is the missing piece from the current observations
- [function] `b-tree-storage-engine/btree.py:_serialize_leaf` — Trace how serialized size grows with each key-value pair and compare against `page_size` to understand overflow risk
- [function] `b-tree-storage-engine/btree.py:PageManager.write_page` — The silent truncation at line ~82 (`data[:self.page_size]`) is a latent data-loss risk if splits don't fire correctly
- [general] `btree-internal-node-splits` — Internal nodes presumably split too when promoting a median key would overflow; understanding the full split cascade from leaf to root
- [function] `b-tree-storage-engine/btree.py:PageManager.allocate_page` — How new pages are provisioned during splits, including free-list reuse

## Beliefs

- `leaf-serialized-overhead-is-7-bytes` — Every leaf page has a fixed 7-byte overhead (3B header + 4B sibling pointer) before any key-value entries
- `leaf-entry-cost-is-variable` — Each leaf entry costs `4 + len(key_bytes) + len(value_bytes)` bytes, so max keys per page depends on actual key/value sizes, not just a fixed count
- `write-page-silently-truncates` — `PageManager.write_page` truncates data exceeding `page_size` without raising an error, making correct split timing essential to avoid data loss
- `page-size-default-is-4096` — `PageManager` defaults to 4096-byte pages, matching typical filesystem block size

