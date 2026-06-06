# Belief-Driven Bug Fixes: From Analysis to Code Changes

**Date:** 2026-06-06

---

## Summary

Used the ddia-expert belief network to identify, file, fix, and verify 8 correctness bugs across the ddia-implementations repository. The fixes triggered cascading belief updates: 8 gating observations were retracted, 8 previously-blocked beliefs became IN, and 31 derived beliefs (including the depth-7 `system-converges-on-permanent-dark-failure`) went OUT as their foundations were removed.

This is a demonstration of the belief-driven development loop: code analysis produces beliefs, beliefs identify bugs, fixes retract the bug observations, and retraction cascades update the entire reasoning network.

---

## Method

### 1. Identifying candidates

`reasons list-gated` returned 32 observations that each blocked a derived belief. Each gate represents a concrete code observation (a bug, missing feature, or scope limitation) that prevents a higher-level claim from being accepted.

### 2. Triage

The 32 gates were triaged into three categories:

| Category | Count | Criterion |
|----------|-------|-----------|
| **Fixable bugs** | 8 | Wrong even for teaching code; straightforward fix |
| **Deliberate scope cuts** | 10 | Intentionally omitted production concerns (concurrency, async networking, persistence) |
| **Gray area** | 14 | Could go either way depending on how production-realistic the implementations should be |

### 3. Fix, retract, observe cascades

Each fix followed the same pattern:
1. Read the gating belief to understand the exact code observation
2. Read the source file to confirm the observation is accurate
3. Apply the minimal fix
4. Run existing tests to verify no regressions
5. Retract the gating belief in reasons.db
6. Observe which beliefs cascade to OUT and which become IN

---

## The 8 Fixes

### Fix 1: Bitcask delete-before-rename (Issue #1)

**Gate:** `delete-before-rename-ordering`
**Files:** `hash-index-storage/bitcask.py`, `log-structured-hash-table/bitcask.py`

Both Bitcask implementations called `os.remove()` on old segment files before `os.rename()` on the compacted replacement. A crash between these operations leaves no valid data file on disk.

**Fix:** Swapped the order — rename first (atomic on POSIX), then delete old files.

**Cascade:** This single retraction took out 31 derived beliefs, including the entire depth-7 chain up to `system-converges-on-permanent-dark-failure`. The chain's argument was: crash-unsafe compaction → no crash-safe storage → no self-healing → monotonic degradation → no stable regime → undetectable failure → permanent dark failure. Removing one premise at the base collapsed the whole structure.

**Unblocked:** `bitcask-compaction-preserves-state`

### Fix 2: Fencing token stale after expiry (Issue #2)

**Gate:** `client-holds-stale-token-after-lock-expiry`
**File:** `fencing-tokens/fencing_tokens.py`

`Client._held_tokens` was never cleared when a lock expired. The client would retain and potentially use a token whose corresponding lock had been acquired by another client.

**Fix:** Added an optional `current_time` parameter to `write_to_resource()` and `get_token()`. When provided, expired tokens are removed from `_held_tokens` and the operation returns an error. Backwards-compatible: without `current_time`, the server-side fencing check still protects against stale writes.

**Unblocked:** `fencing-provides-linearizable-writes`

### Fix 3: WAL BEGIN record type (Issue #3)

**Gate:** `wal-no-begin-marker`
**File:** `write-ahead-log/wal.py`

The WAL had no BEGIN record type, making it impossible during replay to distinguish a standalone PUT from the first record of a multi-record batch. This was the structural root cause of why COMMIT markers could not enforce atomicity.

**Fix:** Added `OP_BEGIN = 5` and emitted a BEGIN record at the start of each batch in `append_batch()`.

**Unblocked:** `wal-batch-replay-provides-atomicity`

### Fix 4: Multi-leader CUSTOM_MERGE validation (Issue #4)

**Gate:** `multi-leader-custom-merge-requires-merge-fn`
**File:** `multi-leader-replication/multi_leader.py`

Constructing a `MultiLeaderCluster` with `CUSTOM_MERGE` strategy and `merge_fn=None` was silently accepted, deferring the error to runtime when the first conflict occurred.

**Fix:** Added a `ValueError` in `__init__()` if strategy is `CUSTOM_MERGE` and `merge_fn` is `None`.

**Unblocked:** `multi-leader-convergence-reliable-across-topologies`

### Fix 5: PBFT non-deterministic digest (Issue #5)

**Gate:** `pbft-default-str-fragile`
**File:** `byzantine-fault-tolerance/pbft.py`

The PBFT digest function used `json.dumps(request, sort_keys=True, default=str)`. The `default=str` fallback silently converts non-serializable objects to their `str()` representation, which is not deterministic across replicas.

**Fix:** Removed `default=str`. Non-serializable types now raise `TypeError` immediately rather than silently producing divergent digests.

**Unblocked:** `pbft-view-change-safety-holds-with-stable-digests`

### Fix 6: SSTable sort order enforcement (Issue #6)

**Gate:** `sstable-sorted-order-caller-responsibility`
**File:** `sstable-and-compaction/sstable.py`

`SSTableWriter.add()` did not enforce sorted key order. Out-of-order keys silently corrupted binary search lookups.

**Fix:** Added `_last_key` tracking and a `ValueError` if the new key is not greater than the previous.

**Unblocked:** `sstable-point-lookup-correct-under-sort-invariant`

### Fix 7: WAL replay atomicity (Issue #7)

**Gate:** `wal-replay-no-atomicity-check`
**File:** `write-ahead-log/wal.py`

`replay()` did not verify that batch operations had a corresponding COMMIT record. Partial batches from a crash were replayed as if committed.

**Fix:** Updated `replay()` to track batch state: buffer records inside BEGIN/COMMIT pairs, flush the buffer on COMMIT, discard on EOF. Individual records outside batches pass through directly.

**Unblocked:** `wal-is-reliable-source-of-truth`

### Fix 8: B-tree page size persistence (Issue #8)

**Gate:** `page-manager-no-page-size-on-disk`
**File:** `b-tree-storage-engine/btree.py`

The page size was not persisted in the data file. Reopening with a different `page_size` value silently corrupted all page reads.

**Fix:** Stored page_size at a fixed offset within metadata page 0 (after the existing 5-field metadata, avoiding changes to META_FMT). On reopen, validates the stored size matches the requested size. Legacy files (stored_page_size == 0 due to zero-padding) are migrated transparently.

**Unblocked:** `btree-page-alignment-enables-corruption-recovery`

---

## Cascade Impact

### By the numbers

| Metric | Count |
|--------|-------|
| Gating beliefs retracted | 8 |
| Additional derived beliefs that went OUT | 33 |
| Beliefs that became IN (unblocked) | 8 |
| Total belief status changes | 49 |

### The depth-7 collapse

The most dramatic cascade came from retracting `delete-before-rename-ordering`. This premise supported `compaction-lacks-crash-safety-across-implementations`, which fed into `storage-crash-recovery-has-no-safe-path`, which was a load-bearing antecedent for multiple depth-3+ beliefs. The entire chain from depth 0 to the depth-7 `system-converges-on-permanent-dark-failure` went OUT.

This illustrates a property of the truth maintenance system: dramatic conclusions built on conjunctive (ALL-of) justifications are fragile. Removing any single antecedent collapses the whole chain. The system's suggestion notes (e.g., "if any single premise is sufficient, restore with `--any`") offer a path to make these robust by switching to disjunctive justifications where appropriate.

### Remaining gates

After the 8 retractions, 24 gating beliefs remain. These fall into:

- **Deliberate scope cuts (10):** Concurrency safety (`compact-no-concurrency-safety`, `gc-not-thread-safe`, `consistent-hash-ring-not-thread-safe`, `flush-clears-then-appends`, `lsm-no-concurrency-control`), synchronous simulation (`distributed-protocols-simulate-synchronous-delivery`, `lamport-send-delivers-synchronously`), missing features that would change the teaching focus (`bloom-filter-not-integrated`, `mvcc-no-persistence`, `dynamo-no-delete-support`).

- **Gray area (12):** Observations that are accurate and fixable but involve judgment about scope — `bitcask-compact-not-crash-safe` (partial fix from our rename ordering), `btree-wal-replays-without-commit` (the B-tree's own WAL, not the standalone one we fixed), `range-scan-no-cycle-guard`, `wal-truncate-not-crash-safe`, `wal-seq-unused-in-recovery`, `lsm-wal-no-crc`, `schema-registry-no-compat-enforcement`, `cdc-consumer-position-is-volatile`, `event-store-persist-no-durability`, `2pc-timeout-unused`, `dynamo-hints-single-homed`, `orset-no-causal-context`.

- **Platform/environment (2):** `macos-fsync-not-durable-without-fullfsync`, `wal-no-directory-fsync`.

---

## Observations

### The belief network as a bug tracker

The gating mechanism turned out to be an effective bug identification tool. Each gate is a concrete code observation with a clear "if this were fixed, then X claim could stand." This gives more actionable structure than a generic bug list — every fix has a known downstream consequence in the reasoning network.

### Conjunctive fragility

The ALL-of justification pattern means that deep derived beliefs are only as strong as their weakest antecedent. `system-converges-on-permanent-dark-failure` sounded alarming but depended on every one of ~50 premises being true simultaneously. Fixing one premise at depth 0 collapsed 7 layers of reasoning. This suggests the derive step should be more conservative about building deep conjunctive chains, or should flag when a conclusion rests on a long ALL-of chain that could be broken by a single fix.

### Fix effort vs. cascade impact

The relationship between fix effort and cascade impact was nonlinear:

| Fix | Lines changed | Beliefs affected |
|-----|--------------|-----------------|
| Bitcask rename ordering | ~20 lines | 32 beliefs (1 IN, 31 OUT) |
| Multi-leader validation | 2 lines | 2 beliefs (1 IN, 1 OUT) |
| PBFT digest | 1 line | 2 beliefs (1 IN, 1 OUT) |

The rename fix was small but sat at the base of the deepest reasoning chain. This suggests prioritizing fixes by their position in the belief dependency graph, not just by code complexity.
