# File: merkle-tree/merkle_tree.py

**Date:** 2026-05-29
**Time:** 11:58

## Purpose

This file implements a **Merkle tree** — a hash tree used for efficient data integrity verification and anti-entropy synchronization. It's a reference implementation for the concept covered in *Designing Data-Intensive Applications*, where Merkle trees appear in the context of detecting inconsistencies between replicas (e.g., Dynamo-style storage systems use them for anti-entropy repair).

The file owns three responsibilities:
1. **Core Merkle tree** (`MerkleTree`) — array-backed binary hash tree with O(log N) updates, inclusion proofs, and tree-diff
2. **Key-range overlay** (`KeyRangeMerkleTree`) — adapts the core tree for key-value anti-entropy, the primary DDIA use case
3. **Streaming construction** (`MerkleTreeBuilder`) — collects leaves incrementally before building

## Key Components

### Constants and Helpers

- **`EMPTY_HASH`** — SHA-256 of empty bytes. Used as the hash for padding leaves and empty trees. This is the sentinel that fills out the tree to a power-of-2 size.
- **`_sha256(data)`** — Thin wrapper returning the hex digest. Every hash in the tree flows through this.
- **`_next_power_of_2(n)`** — Rounds up to ensure the tree is a complete binary tree. This is what makes the array-backing work — a complete binary tree has a known structure that maps cleanly to array indices.

### `MerkleNode`

A dataclass for on-demand tree traversal. Not used internally for storage — the tree is array-backed. Nodes are constructed lazily by `_build_node()` when callers need an object graph (e.g., the `root` property). The `index` field is only meaningful for leaf nodes.

### `MerkleProof`

Carries an inclusion proof: the leaf hash, its index, the sibling hashes along the path to root (each tagged with `"left"` or `"right"` to indicate which side the sibling sits on), and the expected root hash. This is everything a verifier needs without access to the full tree.

### `MerkleTree`

The core implementation. Uses a **flat array** (`self._hashes`) with implicit parent-child indexing:
- Parent of index `i`: `(i - 1) // 2`
- Children of index `i`: `2*i + 1` (left), `2*i + 2` (right)
- Leaves start at index `self._padded_size - 1`

Key methods:

| Method | Contract |
|--------|----------|
| `__init__(data_blocks)` | Builds tree bottom-up in O(N). Pads to power-of-2 with `EMPTY_HASH`. |
| `update_leaf(index, new_data)` | Mutates one leaf and walks up to root recomputing hashes. O(log N). |
| `get_proof(index)` | Returns a `MerkleProof` by collecting sibling hashes from leaf to root. |
| `verify_proof(data, proof)` | Static method. Recomputes the root hash from leaf data + siblings; returns `True` if it matches. |
| `diff(other)` | Returns indices of differing leaves between two same-sized trees. Prunes matching subtrees. |
| `to_dict()` / `from_dict()` | Serialization round-trip. Data blocks are hex-encoded. |

### `KeyRangeMerkleTree`

Wraps `MerkleTree` for the anti-entropy use case. Sorts key-value pairs by key, encodes each as `"key:value"` bytes, and delegates to the core tree. The `diff_keys()` method translates leaf indices back to keys — this is what a replica would call to find which keys are out of sync.

### `MerkleTreeBuilder`

Trivial accumulator. Collects leaves via `add_leaf()`, then calls `MerkleTree(self._leaves)` on `build()`. Useful when data arrives incrementally.

## Patterns

**Array-backed binary tree.** Instead of pointer-based nodes, the tree stores all hashes in a flat list with implicit index arithmetic. This is cache-friendly and avoids object overhead. The `MerkleNode` class exists only for external consumers who want a tree-shaped view — it's never used internally.

**Power-of-2 padding.** The tree always pads to a complete binary tree using `EMPTY_HASH`. This simplifies the index math but means the diff operation must filter out padding indices afterward (the `max_idx` filter in `diff()`).

**Bottom-up construction.** Leaves are hashed first, then internal nodes are computed from `leaf_start - 1` down to index 0. This is the standard O(N) Merkle tree build.

**Hash chaining.** Internal nodes hash the *concatenation of the hex strings* of their children: `sha256((left_hex + right_hex).encode())`. This is a common but specific choice — the hex strings are concatenated as text, then encoded to bytes for hashing.

## Dependencies

**Imports:** Only `hashlib` and stdlib typing/dataclass — no external dependencies. This is a pure data structure with no I/O.

**Imported by:** Two test files (`test_merkle.py`, `test_merkle_tree.py`), and presumably used by anti-entropy components in the broader DDIA implementations (e.g., could plug into `read-repair` or `leaderless-replication`).

## Flow

### Construction
1. Caller provides `data_blocks: List[bytes]`
2. `_next_power_of_2` determines padding size
3. Leaf hashes fill indices `[padded_size - 1, 2*padded_size - 2]`, with `EMPTY_HASH` for padding
4. Internal nodes compute bottom-up from `padded_size - 2` down to `0`
5. `self._hashes[0]` is the root hash

### Proof generation and verification
1. `get_proof(index)` walks from leaf to root, collecting the sibling hash and its side (`"left"`/`"right"`) at each level
2. `verify_proof(data, proof)` starts from `sha256(data)`, then for each sibling applies: `sha256(left + right)` with the sibling on the correct side
3. If the recomputed hash equals `proof.root_hash`, the data is verified

### Diff (anti-entropy)
1. Starts at root (index 0)
2. If hashes match, the entire subtree is identical — prune
3. If hashes differ and we're at a leaf, record the index
4. Otherwise recurse into both children
5. Filter out padding indices at the end

This is the key efficiency gain: diffing two trees with N leaves where K differ is O(K log N) instead of O(N).

## Invariants

- **Power-of-2 leaf count.** `_padded_size` is always a power of 2. All index arithmetic depends on this.
- **Consistent hashing.** Updating a leaf via `update_leaf` walks the full path to root — no lazy invalidation. The root hash is always current.
- **Same-size diff.** `diff()` raises `ValueError` if the trees have different `_padded_size`. This means you can only diff trees built from the same key-range partition.
- **Sorted keys in `KeyRangeMerkleTree`.** The constructor sorts pairs by key. Two trees with the same key-value pairs but different insertion order will produce the same root hash.
- **Leaf index bounds.** `get_leaf`, `update_leaf`, and `get_proof` all validate `index ∈ [0, _leaf_count)` and raise `IndexError` otherwise. This distinguishes real leaves from padding.

## Error Handling

Straightforward — the code validates at boundaries and raises stdlib exceptions:

- **`IndexError`** from `get_leaf`, `update_leaf`, `get_proof` when the leaf index is out of range
- **`ValueError`** from `diff` when trees have different padded sizes
- **`ValueError`** (implicit, from `list.index()`) in `KeyRangeMerkleTree.get_key_proof` and `update_key` if the key doesn't exist — this is a latent issue since `list.index()` raises `ValueError` with a generic message rather than a domain-specific one

No errors are swallowed. `verify_proof` returns a boolean rather than raising on mismatch.

## Topics to Explore

- [file] `merkle-tree/test_merkle.py` — Covers edge cases in proof verification, diff behavior, and the key-range wrapper
- [function] `merkle-tree/merkle_tree.py:_diff_recursive` — The subtree-pruning logic that makes Merkle diffs efficient; worth tracing with a concrete example
- [general] `anti-entropy-repair-protocol` — How `KeyRangeMerkleTree.diff_keys` would be used in a Dynamo-style replica sync loop (connect this to `read-repair` and `leaderless-replication`)
- [function] `merkle-tree/merkle_tree.py:verify_proof` — The static verification path is the trust boundary; worth understanding how the left/right sibling ordering prevents second-preimage attacks
- [general] `hash-concatenation-scheme` — The choice to hash hex-string concatenation (`sha256((hex_a + hex_b).encode())`) vs. raw byte concatenation has security and performance implications worth examining

## Beliefs

- `merkle-tree-array-indexing` — The tree uses 0-indexed implicit array layout where children of node `i` are at `2i+1` and `2i+2`, and leaves start at index `padded_size - 1`
- `merkle-diff-prunes-matching-subtrees` — `diff()` short-circuits recursion when subtree root hashes match, making the diff O(K log N) for K differing leaves rather than O(N)
- `merkle-diff-requires-same-padded-size` — Two trees can only be diffed if they have the same `_padded_size`; the method raises `ValueError` otherwise
- `key-range-merkle-sorts-by-key` — `KeyRangeMerkleTree` sorts input pairs by key before building, so two trees with identical key-value content always produce the same root hash regardless of insertion order
- `merkle-internal-nodes-hash-hex-concatenation` — Internal node hashes are computed as `sha256((left_hex + right_hex).encode())`, concatenating child hex strings as text before encoding to bytes

