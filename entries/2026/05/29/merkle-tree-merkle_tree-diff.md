# Function: diff in merkle-tree/merkle_tree.py

**Date:** 2026-05-29
**Time:** 11:48

## `MerkleTree.diff`

### Purpose

`diff` identifies which leaf positions differ between two Merkle trees. This is the core operation that makes Merkle trees useful for **anti-entropy** in distributed systems: two replicas each build a tree over their data, then diff the roots. If roots match, all data is identical — no further communication needed. If they don't, the diff drills down to find exactly which leaves diverged, transferring only O(log N) hashes instead of comparing every record.

### Contract

**Preconditions:**
- Both trees must have the same `_padded_size` (i.e., the same power-of-2 array dimension). This is enforced with a `ValueError`.
- Both trees must be validly constructed — internal node hashes must reflect their children.

**Postconditions:**
- Returns a list of leaf indices (0-based, in the original data's index space) where the two trees disagree.
- Only real leaf indices are returned — padding slots (those beyond both trees' actual leaf counts) are excluded.

**Invariant:** If `self.root_hash == other.root_hash`, the result is always an empty list. The recursive helper short-circuits at the root and never descends.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `other` | `MerkleTree` | The tree to compare against. Must have been constructed with the same number of padded slots (same power-of-2 rounding). |

Note: two trees can have different `_leaf_count` values and still be diffable, as long as their `_padded_size` matches. For example, a tree with 5 leaves and one with 7 leaves both pad to 8, so they're comparable. The extra slots in the 5-leaf tree will have `EMPTY_HASH`, and those positions will show up as diffs if the 7-leaf tree has data there.

### Return Value

`List[int]` — indices into the original data blocks where the two trees disagree. Sorted in ascending order (as a consequence of the left-to-right DFS in `_diff_recursive`). The caller is responsible for interpreting what the indices mean — `diff` doesn't return the actual data, just the positions.

### Algorithm

1. **Guard:** Reject trees with different `_padded_size` — their array layouts are incompatible.

2. **Recursive descent** via `_diff_recursive(other, 0, result)`, starting at the root (index 0):
   - If `self._hashes[idx] == other._hashes[idx]`, the entire subtree matches. Return immediately — this is the pruning that gives O(k log N) performance where k is the number of differing leaves.
   - If `idx` is at or past `leaf_start` (i.e., `_padded_size - 1`), we've hit a leaf. Append the leaf's logical index (`idx - leaf_start`) to `result`.
   - Otherwise, recurse into both children: `2*idx + 1` (left) and `2*idx + 2` (right).

3. **Filter padding:** The recursion may report diffs in padding positions (indices ≥ both `_leaf_count` values). The final list comprehension strips these out using `max(self._leaf_count, other._leaf_count)` as the upper bound. It uses `max`, not `min` — if tree A has 5 leaves and tree B has 7, positions 5 and 6 are real diffs (B has data that A doesn't), so they should be reported.

### Side Effects

None. The method is purely read-only — it doesn't mutate either tree. The `result` list is local to the call.

### Error Handling

- **`ValueError`**: Raised when `_padded_size` differs between trees. This is the only explicit error. The method does not catch or swallow any exceptions.
- No bounds checking on array access within `_diff_recursive` — this is safe because both trees have identically-sized `_hashes` arrays (guaranteed by the `_padded_size` check) and the recursion stays within `[0, 2*_padded_size - 2]`.

### Usage Patterns

Typical use is through `KeyRangeMerkleTree.diff_keys`, which translates leaf indices back to key names:

```python
tree_a = KeyRangeMerkleTree(replica_a_pairs)
tree_b = KeyRangeMerkleTree(replica_b_pairs)
stale_keys = tree_a.diff_keys(tree_b)  # internally calls self._tree.diff(other._tree)
```

Caller obligation: the two trees must represent the same key-space partitioned identically. If replicas use different sort orders or different key sets, the positional indices are meaningless. `KeyRangeMerkleTree` handles this by sorting keys before construction.

### Dependencies

- `_diff_recursive`: private helper that does the actual traversal. Accesses `_hashes` arrays on both trees and `_padded_size` on `self`.
- The array layout convention: internal nodes at `[0, padded_size-2]`, leaves at `[padded_size-1, 2*padded_size-2]`, children of node `i` at `2i+1` and `2i+2`. This is a standard implicit binary heap layout.

### Untyped Assumptions

- The two trees must represent comparable data sets in the same positional order. Nothing in the type system prevents diffing a tree of user records against a tree of transaction logs.
- `_leaf_count` is assumed to be ≤ `_padded_size` on both trees. Violated values would cause the filter to include padding slots or exclude real leaves.
- The `max` in the filter assumes that a leaf index valid in *either* tree represents real data. If one tree has fewer leaves, those positions in the shorter tree are `EMPTY_HASH` padding — but the diff correctly flags them because the other tree has actual data there.

---

## Topics to Explore

- [function] `merkle-tree/merkle_tree.py:_diff_recursive` — The recursive core that prunes matching subtrees; understanding it reveals the O(k log N) efficiency guarantee
- [function] `merkle-tree/merkle_tree.py:KeyRangeMerkleTree.diff_keys` — Shows how raw index diffs are translated into meaningful key-level diffs for anti-entropy
- [general] `merkle-tree-anti-entropy-in-dynamo` — How Dynamo-style systems use Merkle tree diffs to synchronize replicas, and why positional alignment of keys matters
- [file] `leaderless-replication/test_dynamo.py` — Tests showing how Merkle diffs integrate into a leaderless replication protocol for read repair
- [general] `b-plus-tree-leaf-chains` — Compare Merkle tree's O(k log N) diff with sequential scan approaches; understand when each structure is preferable for anti-entropy

---

## Beliefs

- `merkle-diff-requires-same-padded-size` — `diff` raises `ValueError` if the two trees have different `_padded_size`, meaning both must round to the same power of 2
- `merkle-diff-filters-padding` — The diff result excludes padding indices beyond `max(self._leaf_count, other._leaf_count)`, so only real data positions are returned
- `merkle-diff-is-pure` — `diff` and `_diff_recursive` are read-only; they mutate neither tree and produce no side effects
- `merkle-diff-order-is-ascending` — Differing leaf indices are returned in ascending order as a consequence of left-to-right DFS traversal
- `merkle-diff-asymmetric-leaf-counts` — Two trees with different `_leaf_count` but the same `_padded_size` can be diffed; positions where one tree has data and the other has `EMPTY_HASH` padding are reported as diffs

