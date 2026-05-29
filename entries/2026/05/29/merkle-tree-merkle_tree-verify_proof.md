# Function: verify_proof in merkle-tree/merkle_tree.py

**Date:** 2026-05-29
**Time:** 08:11

# `MerkleTree.verify_proof`

## Purpose

`verify_proof` is a static method that checks whether a piece of data was included in a Merkle tree, given only the data itself and a `MerkleProof`. It reconstructs the root hash by walking the proof's sibling chain from leaf to root, then compares the result against the expected root hash stored in the proof. This is the core verification primitive that makes Merkle trees useful — it lets you confirm membership without having the entire tree, using only O(log N) hashes.

Being a `@staticmethod` is intentional: verification requires no tree instance, just the data and the proof. This means a verifier doesn't need to hold or reconstruct the full tree — a property that matters in distributed systems where one node sends a proof and another validates it.

## Contract

**Preconditions:**
- `data` is the exact bytes that were originally inserted as a leaf.
- `proof` is a well-formed `MerkleProof` whose `siblings` list is ordered bottom-up (leaf → root), matching what `get_proof` produces.

**Postconditions:**
- Returns `True` if and only if hashing `data` produces `proof.leaf_hash` and the reconstructed root matches `proof.root_hash`.
- No state is mutated.

**Invariant:** The method is deterministic — same inputs always produce the same result.

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `data` | `bytes` | The raw data block to verify membership for. Must be the exact bytes originally inserted; a single bit difference causes a hash mismatch. |
| `proof` | `MerkleProof` | An inclusion proof containing the expected leaf hash, the sibling hashes with their positions, and the expected root hash. |

**Edge cases:**
- If `proof.siblings` is empty (a single-leaf tree), the method just checks `hash(data) == leaf_hash == root_hash`.
- If `data` is `b""`, it still works — `_sha256(b"")` produces a valid hash. However, `EMPTY_HASH` is used for padding leaves in the tree, so an empty-bytes proof could collide with padding if the tree wasn't careful about distinguishing real vs. padding leaves.

## Return Value

`bool` — `True` if the proof is valid, `False` otherwise. The caller gets a clean yes/no with no exceptions to handle under normal operation.

## Algorithm

1. **Hash the data:** Compute `_sha256(data)` to get the candidate leaf hash.
2. **Check leaf match:** If this hash doesn't match `proof.leaf_hash`, return `False` immediately. This catches the case where the caller passed wrong data without needing to walk the whole proof.
3. **Walk the sibling chain:** For each `(sibling_hash, direction)` pair, bottom-up:
   - If the sibling is on the **left**, the sibling was the left child and current is the right child: compute `hash(sibling || current)`.
   - If the sibling is on the **right**, current is the left child: compute `hash(current || sibling)`.
   - This mirrors how `get_proof` records siblings — the `direction` indicates where the *sibling* sits, not where the current node sits.
4. **Check root match:** After processing all siblings, `current` should equal the root hash. Return the comparison result.

## Side Effects

None. Pure function — no mutations, no I/O, no state changes.

## Error Handling

The method raises no exceptions by design. If `proof.siblings` contains malformed data (e.g., a direction that is neither `"left"` nor `"right"`), the `else` branch silently treats it as `"right"`. There's no validation of the direction string — an assumption that the proof was produced by `get_proof`, which only emits `"left"` or `"right"`.

If `data` is not `bytes`, `_sha256` will raise a `TypeError` from `hashlib.sha256()`.

## Usage Patterns

```python
tree = MerkleTree([b"alice", b"bob", b"charlie"])
proof = tree.get_proof(1)  # proof for "bob"

# Verifier (possibly on a different machine) checks:
assert MerkleTree.verify_proof(b"bob", proof)      # True
assert not MerkleTree.verify_proof(b"eve", proof)   # False — wrong data
```

The typical caller obligation is to obtain a trustworthy `root_hash` through an independent channel (e.g., a signed block header in a blockchain, or a replicated state in a distributed database). The proof itself is untrusted — verification works precisely because a forger can't produce sibling hashes that reconstruct the correct root without breaking SHA-256.

## Dependencies

- `_sha256(data: bytes) -> str` — the module-level SHA-256 wrapper using `hashlib`.
- `MerkleProof` — the dataclass holding `leaf_hash`, `siblings`, and `root_hash`.

No external dependencies beyond the standard library.

## Notable Assumptions

1. **String concatenation of hex digests:** The method concatenates hex-encoded hash strings and then calls `.encode()` to get bytes before hashing. This matches how `__init__` builds internal nodes (`(left_hash + right_hash).encode()`), so verification and construction are consistent. But it means hashes are computed over 128-character ASCII hex strings rather than raw 32-byte digests — a performance trade-off (4x the data per hash call).

2. **Direction string is never validated:** Any value other than `"left"` falls into the `else` branch. A typo like `"rigth"` would silently be treated as right-concatenation.

3. **The proof's `root_hash` is trusted as the ground truth.** The method tells you "this data matches *this* root," not "this data is in *the* tree." The caller must independently verify that `proof.root_hash` is the authentic root.

---

## Topics to Explore

- [function] `merkle-tree/merkle_tree.py:get_proof` — The counterpart that constructs the proof this method verifies; understanding the sibling ordering is essential
- [function] `merkle-tree/merkle_tree.py:MerkleTree.__init__` — How internal node hashes are computed (must match the reconstruction logic in `verify_proof`)
- [file] `merkle-tree/test_merkle.py` — Test coverage showing proof verification against valid and invalid data
- [general] `merkle-proof-security-model` — How Merkle proofs provide O(log N) verification and what trust assumptions the verifier must satisfy (authentic root hash)
- [function] `merkle-tree/merkle_tree.py:diff` — The anti-entropy use case: comparing two trees to find divergent leaves, which is the DDIA motivation for Merkle trees (Dynamo-style repair)

## Beliefs

- `verify-proof-is-pure-static` — `verify_proof` is a static method with no side effects; it requires no tree instance and can run on a different machine than the one that built the tree
- `verify-proof-hash-consistency` — `verify_proof` concatenates hex-encoded hash strings and encodes to bytes before hashing, matching the exact same scheme used in `__init__` to build internal nodes
- `verify-proof-direction-unvalidated` — The `direction` field in proof siblings is not validated; any value other than `"left"` is silently treated as `"right"`
- `verify-proof-root-trust` — `verify_proof` confirms data matches a given root hash but does not authenticate the root itself; callers must obtain a trusted root through an independent channel
- `merkle-hashes-over-hex-not-bytes` — Internal node hashes are computed over concatenated 64-char hex strings (128 ASCII bytes) rather than raw 32-byte SHA-256 digests, a 4x overhead per hash input

