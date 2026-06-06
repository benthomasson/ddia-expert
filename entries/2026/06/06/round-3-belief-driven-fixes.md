# Round 3: 16 Fixes from Gating Belief Analysis

**Date:** 2026-06-06

---

## How We Found the Issues

After two rounds of fixes (14 bugs total, 58 derived beliefs collapsed), we asked: what issues remain fixable? The method was the same as rounds 1 and 2 — query the belief network for gating observations — but the scope was wider. Round 1 fixed 8 clear bugs. Round 2 fixed 6 more. This round took on 16 fixes that earlier rounds had classified as "gray area" or "deliberate scope cuts," because the cumulative evidence from the belief network made the case that these gaps genuinely undermined the teaching value of the implementations.

### The query

`reasons list-gated` returned 22 remaining observations that each blocked a positive derived belief from becoming IN. We triaged these into two groups:

| Category | Count | Examples |
|----------|-------|---------|
| **Fixable** | 16 | Missing fsync, no checksums, no thread safety, unsafe crash recovery |
| **Design-scope** | 6 | Synchronous delivery model, Dynamo missing tombstones/distributed hints, full LSM concurrency, 2PC async timeouts, macOS F_FULLFSYNC |

The 6 design-scope items would require architectural changes that go beyond improving the reference implementations. The 16 fixable items fell into five natural categories matching the recurring failure patterns we'd documented in the "lessons from the belief network" entry.

### Why these weren't fixed earlier

Rounds 1 and 2 prioritized clear correctness bugs — wrong operation ordering, missing validation, protocol violations. The remaining 16 fixes are hardening measures: adding durability guarantees that were absent, integrity checks that were missing, and safety properties that were silently assumed. They're the difference between "works in the happy path" and "works when things go wrong."

---

## The 16 Fixes in 5 Groups

### Group A — Fsync/Durability (5 fixes)

Pattern: `flush()` without `os.fsync()` means data reaches the OS page cache but not the disk. A power loss after flush but before the OS writes back loses the data silently.

| # | File | Function | What changed |
|---|------|----------|-------------|
| A1 | `event-sourcing-store/event_store.py` | `_persist_event()` | Added `f.flush(); os.fsync(f.fileno())` after JSON write |
| A2 | `log-structured-hash-table/bitcask.py` | `_write_record()` | Added `os.fsync()` after existing `flush()` |
| A3 | `log-structured-hash-table/bitcask.py` | `create_hint_files()` | Added `hf.flush(); os.fsync(hf.fileno())` before close |
| A4 | `write-ahead-log/wal.py` | `_rotate()` | Added directory fsync after creating new segment file |
| A5 | `sstable-and-compaction/sstable.py` | `finish()` | Added `flush(); os.fsync()` before close |

A4 uses the directory fsync pattern — `os.open(dir, O_RDONLY)` + `os.fsync(dir_fd)` — because file creation modifies directory metadata. Without it, a crash can leave the directory entry pointing to nothing even though the file's data blocks are on disk.

**Beliefs retracted:** `event-store-persist-no-durability`, `log-structured-bitcask-no-fsync`, `hint-file-no-fsync`, `wal-no-directory-fsync`, `sstable-writer-finish-no-fsync`

**Beliefs unblocked:** `event-sourcing-state-is-fully-reconstructible`, `wal-multi-segment-continuity-is-reliable`

### Group B — Defensive Checks (3 fixes)

| # | File | Issue | Fix |
|---|------|-------|-----|
| B1 | `b-tree-storage-engine/btree.py` | `range_scan()` walks leaf sibling chain with no cycle detection | Added visited-set guard; raises `ValueError` on cycle |
| B2 | `log-structured-merge-tree/lsm.py` | `_flush()` clears memtable before writing SSTable — crash between loses data | Reordered: write SSTable → truncate WAL → clear memtable |
| B3 | `b-tree-storage-engine/btree.py` | WAL replays all valid entries regardless of commit status | Added `COMMIT_PAGE_NUM` sentinel; `recover()` buffers entries and only applies committed sets |

B2 is the subtlest. The original ordering was: (1) clear memtable, (2) write SSTable, (3) truncate WAL. A crash after step 1 but before step 2 loses the memtable contents — they're gone from memory and the WAL hasn't been replayed yet because we're mid-flush. The fix makes the SSTable write happen first, so at every point either the data is in the memtable (recoverable from WAL) or it's in the SSTable.

B3 mirrors the standalone WAL fix from round 1 (Issue #7) but for the B-tree's own WAL. The B-tree WAL writes page-level redo entries with a commit marker at the end. Without checking for the commit marker during recovery, a partial operation (e.g., a node split that wrote the child page but crashed before writing the parent update and commit marker) gets replayed, leaving the tree in an inconsistent state.

**Beliefs retracted:** `range-scan-no-cycle-guard`, `flush-clears-then-appends`, `btree-wal-replays-without-commit`

**Cascade:** Retracting `range-scan-no-cycle-guard` cascaded through `range-scan-lacks-safety-guarantees` and `concurrency-unsafe-on-both-read-and-write-paths` (3 OUT total). The derived belief `btree-range-scan-provides-ordered-access` became IN.

**Beliefs unblocked:** `btree-range-scan-provides-ordered-access`, `lsm-read-path-correct-across-flushes`, `btree-splits-are-crash-safe`

### Group C — Crash Safety (2 fixes)

| # | File | Issue | Fix |
|---|------|-------|-----|
| C1 | `write-ahead-log/wal.py` | `truncate()` rewrites file in place; crash mid-truncate loses both old and new data | Write to temp file → fsync → atomic rename → directory fsync |
| C2 | `hash-index-storage/bitcask.py` | Compaction writes new file but doesn't fsync before close; no directory fsync after rename | Added `merged_file.flush(); os.fsync()` before close; directory fsync after rename |

C1 applies the atomic file replacement pattern: write-to-temp, fsync the temp, rename over the original (atomic on POSIX), fsync the directory. At every point during this sequence, either the old file or the new file is complete on disk. The previous implementation used `open("wb")` which truncates the original file immediately — a crash during the write leaves a partial file.

**Beliefs retracted:** `wal-truncate-not-crash-safe`, `bitcask-compact-not-crash-safe`

**Cascade:** `wal-truncation-is-multi-failure-hazard` went OUT (its ALL-of justification required `wal-truncate-not-crash-safe`).

**Beliefs unblocked:** `wal-truncation-safe-when-linear`, `hash-index-survives-restart`

### Group D — Concurrency Safety (2 fixes)

| # | File | Issue | Fix |
|---|------|-------|-----|
| D1 | `consistent-hashing/consistent_hashing.py` | Ring operations (`add_node`, `remove_node`, `get_node`, `get_nodes`) not thread-safe | Added `threading.Lock`; public/private method split |
| D2 | `snapshot-isolation/mvcc_database.py` | All MVCC operations on shared state not thread-safe | Added `threading.RLock`; wrapped all public methods; fixed internal `self.abort()` → `self._abort()` in `_commit()` |

D1 uses the public/private method pattern: `add_node()` acquires the lock and calls `_add_node()`. Internal methods that need to compose operations (e.g., a remove that checks state before modifying) call the private version directly, avoiding deadlock.

D2 uses `RLock` (re-entrant) because `_commit()` calls `_abort()` internally on conflict detection. With a regular `Lock`, this would deadlock since `commit()` already holds the lock. The `_scan()` method also calls `self.read()` per key — another re-entrant call that requires `RLock`.

**Beliefs retracted:** `consistent-hash-ring-not-thread-safe`, `gc-not-thread-safe`

**Cascade:** `multiple-core-components-assume-single-thread-silently` went OUT (required both `consistent-hash-ring-not-thread-safe` and `gc-not-thread-safe`). This cascaded further to `distributed-correctness-undermined-at-both-layers` (2 more OUT).

**Beliefs unblocked:** `consistent-hash-routing-correct-under-single-thread`, `ssi-isolation-holds-under-single-thread`

### Group E — Checksums/Integrity (4 fixes)

| # | File | Issue | Fix |
|---|------|-------|-----|
| E1 | `log-structured-merge-tree/lsm.py` | WAL has no checksums; corrupt or torn records replayed silently | Added 4-byte CRC32 prefix per record; `replay()` verifies and stops on mismatch |
| E2 | `hash-index-storage/bitcask.py` | No record-level integrity checking | Added CRC32 to record header (matching log-structured variant); verified on read, scan, and compact |
| E3 | `sstable-and-compaction/sstable.py` | No per-entry checksums; corruption propagates silently through compaction | Added 4-byte CRC32 prefix per entry; writer computes over entry payload, reader verifies |
| E4 | `avro-serializer/avro_serializer.py` | Schema registry accepts any schema regardless of backward compatibility | Added `_subjects` dict; `register()` checks backward compatibility against latest version under same subject |

E1 and E3 follow the same pattern: CRC32 computed over the serialized payload, written as a 4-byte big-endian prefix. On read, the payload is re-read and the CRC recomputed. Mismatch means corruption — the reader stops (WAL replay) or raises `IOError` (SSTable read).

E2 changes the hash-index bitcask record format from `[timestamp:8][key_size:4][value_size:4]` (16 bytes) to `[crc:4][timestamp:8][key_size:4][value_size:4]` (20 bytes). CRC covers `key_bytes + val_bytes`. This matches the log-structured-hash-table variant's format, which already had this structure.

E4 is different — it's a semantic integrity check rather than byte-level. The Avro serializer's `check_compatibility()` function already existed but was never called during registration. The fix adds subject tracking and calls `check_compatibility()` against the latest version under the same subject before accepting a new schema.

**Beliefs retracted:** `lsm-wal-no-crc`, `hash-index-no-crc-by-design`, `sstable-no-checksums`, `schema-registry-no-compat-enforcement`

**Cascade:** `sstable-no-checksums` triggered the largest cascade of this round — 8 beliefs went OUT, including `sstable-format-lacks-integrity-and-efficiency-mechanisms`, `integrity-degrades-along-storage-pipeline`, `read-path-unreliable-from-storage-through-derived-systems`, `format-rigidity-prevents-evolutionary-repair`, and `verification-impossible-at-every-layer`. These were deep derived beliefs (depth 3-4) that depended on the SSTable lacking checksums as a key premise.

**Beliefs unblocked:** `lsm-wal-provides-crash-recovery`, `avro-schema-evolution-handles-version-mismatch`

---

## Cascade Impact

### By the numbers

| Metric | Before round 3 | After round 3 | Delta |
|--------|----------------|---------------|-------|
| Beliefs IN | 1,311 | 1,294 | -17 |
| Beliefs OUT | 94 | 111 | +17 |
| Total beliefs | 1,405 | 1,405 | 0 |
| Premises retracted | — | 16 | — |
| Derived beliefs cascaded OUT | — | 13 | — |
| Beliefs unblocked (became IN) | — | 10 | — |

The IN count went down because more derived beliefs cascaded OUT (29 total: 16 premises + 13 derived) than became IN (10 unblocked). This is expected — many of the unblocked beliefs had their own antecedents that are still OUT.

### Cumulative across all 3 rounds

| Metric | Round 1 | Round 2 | Round 3 | Total |
|--------|---------|---------|---------|-------|
| Bugs fixed | 8 | 6 | 16 | 30 |
| Beliefs retracted | 8 | 7 | 16 | 31 |
| Derived beliefs cascaded | 33 | 27 | 13 | 73 |
| Beliefs unblocked | 8 | 8 | 10 | 26 |

The cascade impact per retraction is declining: round 1 averaged 4.1 cascades per retraction, round 2 averaged 3.9, round 3 averages 0.8. This is because rounds 1 and 2 removed load-bearing premises deep in conjunctive chains. By round 3, the remaining beliefs are more isolated — their retraction doesn't collapse long chains because those chains were already dismantled.

### The SSTable cascade

The largest single cascade in round 3 came from retracting `sstable-no-checksums`, which took out 7 derived beliefs in a single cascade. The chain was:

```
sstable-no-checksums (retracted)
  → sstable-format-lacks-integrity-and-efficiency-mechanisms (OUT)
    → sstable-layer-compounds-integrity-and-performance-deficiencies (OUT)
      → integrity-degrades-along-storage-pipeline (OUT)
        → verification-impossible-at-every-layer (OUT)
      → read-path-unreliable-from-storage-through-derived-systems (OUT)
    → binary-formats-rigid-across-entire-storage-stack (OUT)
      → format-rigidity-prevents-evolutionary-repair (OUT)
```

This mirrors the round 1 pattern where `delete-before-rename-ordering` collapsed 31 beliefs — a single premise at the base of a deep conjunctive chain. The difference is that this chain was shallower (max depth 4 vs 7) because rounds 1 and 2 had already trimmed the deepest chains.

---

## Remaining Gating Beliefs

After 3 rounds, 6 gating observations remain that we classified as design-scope:

| Belief | Why not fixed |
|--------|--------------|
| `distributed-protocols-simulate-synchronous-delivery` | Fundamental to the simulation approach — async message delivery would require a different architecture |
| `lamport-send-delivers-synchronously` | Same: synchronous delivery is the pedagogical model |
| `dynamo-no-delete-support` | Missing tombstone/hint mechanisms would be a new feature, not a fix |
| `dynamo-hints-single-homed` | Distributed hinted handoff requires network layer changes |
| `lsm-no-concurrency-control` | Full concurrent LSM (lock-striped memtable, concurrent compaction) is production scope |
| `2pc-timeout-unused` | Async timeout handling requires event-loop or threading changes |

These represent genuine limitations but they're at the boundary between "reference implementation" and "production system." Fixing them would shift the implementations from teaching tools to something closer to working databases, which changes the project's purpose.

---

## Method Retrospective

### What worked

**The gating query as a prioritization tool.** `reasons list-gated` directly surfaces observations that, if fixed, would unblock positive derived beliefs. This is more useful than a generic code review because every fix has a known downstream consequence — you can predict the cascade before applying it.

**Grouping by failure pattern.** Organizing the 16 fixes into 5 categories (fsync, defensive checks, crash safety, concurrency, checksums) made the work systematic. Each group applied a single pattern across multiple implementations, which reduced the chance of inconsistent fixes and made the commit history coherent.

**Test-after-each-group cadence.** Running tests after each group caught issues early. The hash-index bitcask CRC change touched the record format, which affected `_write_record`, `_read_record`, `_scan_data_file`, and `compact` — a format change that required coordinated updates across four methods. Testing after the group caught the `self._dir` vs `self.data_dir` attribute mismatch that would have been harder to debug later.

### What the belief network revealed that code review wouldn't

A traditional code review might find any individual issue on this list. What the belief network adds is the *relationship* between issues. For example:

- `sstable-no-checksums` alone is a known TODO. But the belief network connects it to `integrity-degrades-along-storage-pipeline` (combined with WAL CRC gaps) and then to `verification-impossible-at-every-layer` (combined with missing test coverage for corruption). The network shows that fixing SSTable checksums doesn't just add a feature — it breaks a chain of reasoning that concludes the entire storage stack lacks verifiable integrity.

- `flush-clears-then-appends` alone looks like a minor ordering bug. But the network connects it to `lsm-read-path-correct-across-flushes`, which was gated specifically on the flush ordering. Fixing the ordering doesn't just prevent a crash-time data loss — it validates that the LSM read path correctly sees data across the flush boundary.

These cross-cutting connections are exactly what belief networks capture and code reviews don't. A reviewer sees files; the network sees relationships between invariants.
