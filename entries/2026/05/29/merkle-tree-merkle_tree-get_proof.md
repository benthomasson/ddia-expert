# Function: get_proof in merkle-tree/merkle_tree.py

**Date:** 2026-05-29
**Time:** 11:46

## `MerkleTree.get_proof` — Merkle Inclusion Proof Generator

### Purpose

`get_proof` constructs a **Merkle inclusion proof** (also called an authentication path) for a specific leaf. The proof is the minimal set of sibling hashes needed to recompute the root hash starting from that leaf. A verifier who holds the root hash can use this proof to confirm that a particular data block exists in the tree **without needing the entire tree** — this is the core mechanism behind efficient data verification in systems like certificate transparency logs, BitTorrent, and Dynamo-style anti-entropy protocols.

### Contract

- **Precondition**: `index` must be in `[0, self._leaf_count)`. The tree must be fully constructed (all internal hashes computed).
- **Postcondition**: The returned `MerkleProof` contains exactly `height` sibling entries — one per level from leaf to root (exclusive). Feeding these siblings back through `verify_proof` with the original data will return `True`.
- **Invariant**: The proof is read-only; calling `get_proof` does not mutate the tree.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `index` | `int` | Zero-based index into the *real* leaves (not the padded array). Must satisfy `0 <= index < self._leaf_count`. |

Edge cases: negative indices are explicitly rejected (not treated as Python-style negative indexing). An index equal to `_leaf_count` raises even though the backing array has padded slots beyond it.

### Return Value

Returns a `MerkleProof` dataclass with:

- `leaf_hash`: the SHA-256 hash of the leaf at `index`
- `leaf_index`: the original `index` (echoed back for convenience)
- `siblings`: list of `(hash, direction)` tuples, ordered **bottom-up** (leaf level first, root's children last). `direction` is `"left"` or `"right"`, indicating **where the sibling sits** relative to the current node — this tells the verifier which side to place the sibling hash when concatenating.
- `root_hash`: the tree's root hash at proof-generation time

The caller must understand that the proof is a snapshot — if `update_leaf` is called after proof generation, the proof becomes invalid against the new root.

### Algorithm

1. **Bounds check** — reject out-of-range indices immediately.

2. **Map logical index to array index** — the tree is stored as a flat array in level-order. Leaves start at position `_padded_size - 1`, so leaf `i` lives at array index `_padded_size - 1 + i`.

3. **Collect siblings bottom-up** — walk from the leaf toward the root:
   - Compute the parent: `parent = (arr_idx - 1) // 2` (standard heap-array formula).
   - Determine whether the current node is the left child (`2*parent + 1`) or right child (`2*parent + 2`).
   - Grab the **other** child's hash (the sibling) and record which side it's on.
   - Move up: `arr_idx = parent`.
   - Stop when `arr_idx == 0` (the root has no sibling).

4. **Package the proof** — wrap the leaf hash, siblings list, and current root hash into a `MerkleProof`.

The key insight in step 3: the `direction` label tells the verifier how to reconstruct each internal hash. If the sibling is on the `"right"`, the verifier computes `H(current || sibling)`. If on the `"left"`, it computes `H(sibling || current)`. This matches how `verify_proof` consumes the proof.

### Side Effects

None. This is a pure read operation — no mutations to `_hashes`, `_data`, or any other state.

### Error Handling

- **`IndexError`**: raised if `index < 0` or `index >= self._leaf_count`. The error message includes the valid range.
- No other exceptions are possible assuming the tree was constructed correctly (array indices are always valid given correct `_padded_size`).

### Usage Patterns

Typical usage pairs `get_proof` with `verify_proof`:

```python
tree = MerkleTree([b"alice", b"bob", b"carol"])
proof = tree.get_proof(1)  # proof for "bob"

# Later, possibly on a different machine:
assert MerkleTree.verify_proof(b"bob", proof)
```

Also used in `KeyRangeMerkleTree.get_key_proof`, which translates a key lookup into a leaf index and delegates here.

Caller obligations:
- The caller must retain the original data to verify later (the proof contains hashes, not data).
- If the tree is mutable, the caller should not assume the proof remains valid after `update_leaf`.

### Dependencies

- `_sha256` — the project-local wrapper around `hashlib.sha256` (used indirectly; the proof itself just reads precomputed hashes).
- `MerkleProof` — the dataclass that packages the proof.
- The array-backed tree layout: the entire method depends on the convention that node `i`'s children are at `2i+1` and `2i+2`, and that leaves are padded to the next power of 2.

### Assumptions Not Enforced by the Type System

1. **`index` is treated as a non-negative integer** — Python's `int` permits arbitrary values; only the runtime bounds check catches violations.
2. **The internal `_hashes` array is assumed to be fully populated** — there is no check for empty strings or sentinel values. If the tree were partially constructed, `get_proof` would silently return a proof with invalid hashes.
3. **Padding leaves use `EMPTY_HASH`** — the proof may include `EMPTY_HASH` siblings for trees whose leaf count isn't a power of 2. This is correct but means a verifier could construct a valid proof for "empty" data at a padding index if the bounds check were bypassed.

---

## Topics to Explore

- [function] `merkle-tree/merkle_tree.py:verify_proof` — The counterpart that consumes the proof; understanding the hash-concatenation order convention is essential to understanding why `direction` matters
- [function] `merkle-tree/merkle_tree.py:update_leaf` — Uses the same parent-walk pattern but writes hashes instead of reading siblings; compare the two to see how the array layout supports both operations in O(log N)
- [general] `merkle-proof-security-properties` — Why inclusion proofs work (collision resistance), what they don't prove (exclusion), and how sorted Merkle trees or prefix trees extend them to support non-membership proofs
- [file] `merkle-tree/test_merkle.py` — Verify that proofs round-trip correctly and that edge cases (single-leaf tree, non-power-of-2 leaf counts) are covered
- [function] `merkle-tree/merkle_tree.py:diff` — The anti-entropy use case: how two replicas use root-hash comparison to efficiently find divergent leaves, which is the other major application of Merkle trees in DDIA

---

## Beliefs

- `merkle-proof-sibling-order-bottom-up` — `get_proof` returns siblings ordered from leaf level to root level (bottom-up), and `verify_proof` consumes them in that same order
- `merkle-proof-direction-is-sibling-position` — The `direction` field in each sibling tuple indicates where the **sibling** sits (left or right), not where the current node sits — this controls concatenation order during verification
- `merkle-proof-length-equals-height` — A proof for a tree with `_padded_size = 2^h` contains exactly `h` sibling entries, one per tree level excluding the root
- `merkle-proof-is-snapshot` — Proofs are not invalidated in-place; a proof generated before `update_leaf` will fail verification against the new root hash
- `merkle-get-proof-rejects-padding-indices` — The bounds check uses `_leaf_count` (real leaves), not `_padded_size`, so proofs cannot be generated for padding slots even though they have valid hashes in the array

