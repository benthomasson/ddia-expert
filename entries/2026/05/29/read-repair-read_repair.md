# File: read-repair/read_repair.py

**Date:** 2026-05-29
**Time:** 09:04

# `read-repair/read_repair.py`

## Purpose

This file implements **read repair**, a consistency mechanism used in leaderless (Dynamo-style) replication systems. It's a reference implementation of the concept from Chapter 5 of *Designing Data-Intensive Applications* — specifically the technique where stale replicas are detected and corrected lazily during read operations, rather than requiring a synchronous protocol.

The module owns two responsibilities: (1) quorum-based reads and writes across a set of in-memory replicas, and (2) opportunistic repair of stale replicas discovered during reads.

## Key Components

### `Replica`

A single node holding a versioned key-value store (`key -> (value, version)`). The version acts as a simple scalar clock — `put` accepts a write only if the incoming version is >= the current version. This is the version-gating contract: stale writes are silently rejected (return `False`), which prevents read repair from regressing a replica that has already advanced.

### `ReadRepairStore`

The coordinator that sits in front of N replicas and enforces quorum semantics.

- **`put(key, value)`** — Determines the next version by scanning *all* replicas (not just available ones), then writes to the first W available replicas. Returns the version assigned and which replicas received the write.
- **`get(key)`** — Reads from R available replicas, picks the highest-versioned response, then **pushes that value to any stale replicas among the R it just read from**. This is the read repair. Returns metadata including which replicas were repaired and whether the read was consistent.
- **`anti_entropy_repair(key)`** — A background/manual repair that syncs *all* available replicas to the latest version. This complements read repair by covering replicas that weren't in the read quorum.
- **`get_repair_stats()`** — Exposes counters: total reads, reads that triggered repair, total replicas repaired, and the repair rate (fraction of reads needing repair).

### `InsufficientReplicasError`

Raised when fewer replicas are available than the quorum requirement. This is the only custom exception — it gates both reads and writes.

## Patterns

**Quorum intersection.** The constructor checks `R + W > N` and warns (via `warnings.warn`, not an exception) if the condition isn't met. This is the classic Dynamo quorum rule: if R + W > N, at least one replica in every read quorum must overlap with the write quorum, guaranteeing the read sees the latest write. The warning-not-error design lets you intentionally run with weaker consistency (e.g., R=1, W=1 for availability).

**Lazy repair.** Read repair only fixes the replicas that were *queried* during the read — not all replicas. This is the defining characteristic: it piggybacks consistency healing onto normal read traffic. Replicas that are never read stay stale until anti-entropy runs.

**Version as scalar clock.** Versions are simple incrementing integers, determined by the coordinator scanning all replicas for the current max. This avoids conflicts but assumes a single logical writer (no concurrent coordinators racing on version assignment). In a real system, vector clocks or hybrid logical clocks would replace this.

**First-N selection.** Both `put` and `get` pick the first N available replicas from the list. There's no randomization or load balancing — the selection is deterministic based on replica order. This is a simplification that makes behavior predictable for testing.

## Dependencies

**Imports:** Only `warnings` from the standard library. No external dependencies — this is a self-contained simulation.

**Imported by:** `test_read_repair.py` and `tester_test_read_repair.py`, which exercise the quorum and repair behavior.

## Flow

### Write path
1. Filter to available replicas; raise if fewer than W.
2. Scan **all** replicas (including unavailable — note the code reads from `self.replicas`, not `available`) to find the max existing version for the key.
3. Assign `new_version = max_version + 1`.
4. Write to the first W available replicas.

### Read path
1. Filter to available replicas; raise if fewer than R.
2. Read the key from the first R available replicas.
3. Find the response with the highest version.
4. If any of the R replicas had a stale or missing value, push the best value to them (read repair).
5. Return the value, version, and repair metadata.

### Anti-entropy path
1. Scan all available replicas for the max version.
2. Push that version to any available replica that's behind.

## Invariants

- **Quorum gate**: Every `get` and `put` checks available replica count against the quorum threshold before proceeding. No partial reads or writes.
- **Version monotonicity**: `Replica.put` rejects any version strictly less than the current version. Equal versions are accepted (last-writer-wins for same-version ties).
- **Read repair scope**: Only replicas in the read quorum are repaired — not all replicas. This is intentional and is the key difference from anti-entropy.
- **Version determination reads unavailable replicas**: `put` iterates `self.replicas` (all replicas) to find max version, meaning it reads from replicas marked unavailable. This could be a bug or a deliberate choice to avoid version regression — if a replica was recently marked unavailable but had the latest version, skipping it would assign a duplicate version number.

## Error Handling

- `InsufficientReplicasError` is raised synchronously on both read and write when the quorum can't be met.
- The quorum condition violation (`R + W <= N`) produces a `warnings.warn` — it's informational, not fatal, allowing relaxed quorum configurations.
- `Replica.put` returns `False` for rejected stale writes but this return value is never checked by `ReadRepairStore` — the coordinator trusts that version monotonicity makes redundant writes harmless.

## Topics to Explore

- [file] `leaderless-replication/dynamo.py` — The broader Dynamo-style store that this read repair mechanism would sit inside; compare how quorum reads/writes are handled there
- [function] `read-repair/read_repair.py:put` — The version assignment scans all replicas including unavailable ones — investigate whether this is intentional or a consistency bug
- [file] `hinted-handoff/hinted_handoff.py` — Hinted handoff is the complementary write-path mechanism to read repair; together they form the Dynamo consistency toolkit
- [file] `vector-clocks/vector_clock.py` — How version tracking works with vector clocks vs. the scalar versioning used here
- [general] `anti-entropy-vs-read-repair` — The tradeoffs between lazy read-path repair and background anti-entropy (Merkle tree exchange) for convergence speed and read amplification

## Beliefs

- `read-repair-scope-is-quorum-only` — Read repair in `get()` only fixes replicas that were part of the read quorum, not all replicas; full-cluster repair requires `anti_entropy_repair()`
- `version-scan-includes-unavailable` — `put()` determines the next version by scanning all replicas including those marked unavailable, which prevents version regression but couples writes to unavailable node state
- `replica-put-rejects-older-versions` — `Replica.put()` silently rejects writes with a version strictly less than the stored version, enforcing monotonic version advancement
- `quorum-violation-warns-not-raises` — When `R + W <= N`, the constructor emits a `warnings.warn` rather than raising an exception, allowing intentionally relaxed consistency configurations
- `put-return-value-ignored-by-coordinator` — `ReadRepairStore` never checks the boolean return from `Replica.put()` during read repair or writes, relying on version ordering to make redundant writes no-ops

