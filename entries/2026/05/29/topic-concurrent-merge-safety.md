# Topic: Neither implementation handles concurrent readers during merge; the paper describes an atomic switchover where old files are deleted only after the keydir is updated — trace whether the `compact()` methods maintain this invariant or have a window where reads could fail

**Date:** 2026-05-29
**Time:** 08:56

I don't have read access to the target repo (`/Users/ben/git/ddia-implementations`), so I'll work from the detailed entries already captured in this knowledge base. The entries contain line-by-line analysis of both `compact()` methods.

---

# Concurrent Readers During Bitcask Compaction: The Atomic Switchover Problem

## What the Paper Requires

The original Bitcask paper (Sheehy & Smith, 2010) describes merge as an operation that can run **concurrently with readers**. The design relies on an **atomic switchover** invariant:

1. Write all merged data to new files
2. Update the keydir to point to the new file locations
3. **Only then** delete old files

This ordering guarantee is critical: because readers look up the keydir first (to get a file_id and offset), and then do a positional read on the file, old files must remain readable until the keydir no longer references them. The keydir update is the atomic commit point — before it, readers use old files (which still exist); after it, readers use new files (which are fully written).

In the Erlang implementation, the keydir is a shared ETS table, and updates to individual entries are atomic at the per-key level. Old file handles are reference-counted and closed only when the last reader releases them.

## Neither Implementation Maintains This Invariant

Both implementations are explicitly single-threaded and assume **no concurrent access** during compaction. But even ignoring the threading question, neither one sequences operations in the order the paper requires.

### `hash-index-storage/bitcask.py:compact()` — Operation Sequence

From the entry on this function, the method runs in six phases:

| Phase | What happens | Keydir state | Old files state |
|-------|-------------|--------------|-----------------|
| 1 | Identify immutable files | Points to old files | Exist |
| 2 | Scan immutable files for latest records | Points to old files | Exist |
| 3 | Filter against keydir | Points to old files | Exist |
| 4 | **Write merged files + update keydir in-place** (line ~276) | **Being mutated** — points to mix of old and new | Exist |
| 5 | **Delete old files** (lines 288–290) | Points to new files | **Being deleted** |
| 6 | Rename active file + update keydir for active entries | Fully updated | Deleted |

The **danger window** is in Phase 4: the keydir is updated **entry-by-entry** as each record is written to the merged output. During this phase, some keys point to new merged files while others still point to old immutable files. If a concurrent reader looked up a key whose keydir entry still pointed to an old file, the read would succeed — the old file is still on disk. But this is **not atomic** — there's a period where the keydir is internally inconsistent.

The real problem is between Phases 4 and 5. After Phase 4 completes, all keydir entries for compacted keys point to the new merged files. But the old file handles in `self.file_handles` were **closed at the start of Phase 4** (the entry notes "Closes readers for old immutable files, then writes..."). This means even if a concurrent reader's keydir lookup returned an old file_id, there's no cached reader available for it.

**Crash window**: If the process dies between Phase 5 (deleting old files, line 288) and Phase 6 (renaming the active file, line 297), the old segments are gone but the active file hasn't been renumbered. On recovery, `_rebuild_index` scans `_find_file_ids()` — it would find the active file at its old ID and the merged files at their new IDs, but **data that was only in the deleted old segments and hasn't yet been reflected in the active file rename is permanently lost**.

### `log-structured-hash-table/bitcask.py:compact()` — Operation Sequence

From the companion entry, this method runs in five phases:

| Phase | What happens | Index state | Old files state |
|-------|-------------|-------------|-----------------|
| 1 | Scan frozen segments for live entries | Points to old segments | Exist |
| 2 | Filter against live index | Points to old segments | Exist |
| 3 | **Close active file**, write compacted segment, build new index entries | Active file closed — **reads broken** | Exist |
| 4 | **Update index + delete old segments** (lines 292–295) | Being mutated | **Being deleted** |
| 5 | Rename active segment to highest ID, reopen | Fully updated | Deleted |

This implementation has an even more severe concurrency issue: **Phase 3 closes the active file** before writing the compacted segment. The entry explicitly states:

> "Closes the active file, bumps the segment counter, and opens a new compacted segment file."

During Phase 3, `self._active_file` is closed. Any concurrent `get()` for a key in the active segment would need to open a file handle — and the active path might not match what's expected since the segment counter has been bumped. A concurrent `put()` during this window would write to a closed or wrong file, corrupting state.

Phase 4 is where the index is updated **and** old segments are deleted in the same phase. The entry describes the cleanup order as: update `_index` with new locations, then close cached handles, then delete old segment files and hint files. But there's no barrier between the index update and the file deletion. If a concurrent reader looked up a key, saw the old path in the index (before that entry was updated), and tried to read it — the file might already be deleted.

## The Core Gap: No Reference Counting, No Manifest

Both implementations share the same structural flaw: **old files are deleted based on the assumption that nothing references them, but this assumption is enforced by convention (single-threaded access) rather than mechanism**.

The paper's design handles this through:
1. **Reference-counted file handles** — a file isn't deleted until the last reader closes its handle
2. **Atomic keydir updates per key** — Erlang ETS table writes are atomic
3. **Implicit ordering** — all keydir entries are updated before any old file is unlinked

Neither implementation has any of these. They also lack a **manifest** (as noted in the compaction atomicity entry) — the standard solution used by LevelDB and RocksDB where a manifest file atomically records "replace segments [A, B, C] with [D]", making the delete step safe because recovery reads the manifest rather than discovering segments via directory listing.

## Summary of Read-Failure Windows

| Scenario | `hash-index-storage` | `log-structured-hash-table` |
|----------|---------------------|----------------------------|
| Concurrent `get()` during Phase 4 (keydir update) | Old file handles already closed — read fails | Active file closed — reads for active-segment keys fail |
| Concurrent `get()` between keydir update and file deletion | Might work (file still exists, but no cached handle) | File might already be deleted if index and delete are interleaved |
| Crash between file deletion and active rename | **Data loss** — old segments gone, active not renumbered | **Data loss** — same structural issue |
| Crash after partial file deletion | Some keys lost, no recovery path | CRC helps detect corrupt records but can't recover deleted files |

**Bottom line**: Neither implementation supports concurrent reads during compaction, and the operation ordering in both creates windows where even a crash (not just concurrent access) can cause permanent data loss. The paper's atomic switchover invariant — update keydir fully, then delete old files — is not maintained by either implementation.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:_rebuild_index` — Trace what happens when recovery finds both old and merged files coexisting after a mid-compaction crash, and whether it produces a correct keydir
- [function] `log-structured-hash-table/bitcask.py:_recover` — Whether the recovery path can tolerate the "active segment at wrong ID" state left by a crash between Phases 4 and 5 of compaction
- [general] `manifest-based-compaction` — How LevelDB's MANIFEST makes the old-to-new segment swap atomic and crash-safe, and what a minimal manifest for these implementations would look like
- [function] `log-structured-hash-table/bitcask.py:get` — Trace the read path to see exactly which file handles are used and what happens when a handle is missing or a file doesn't exist
- [general] `reference-counted-file-handles` — The Erlang Bitcask implementation uses reference counting so old segment files stay open until the last reader finishes; compare with the Python implementations' approach of eagerly closing handles during compaction

## Beliefs

- `compact-closes-handles-before-keydir-fully-updated` — In `hash-index-storage/bitcask.py`, cached file readers for old immutable files are closed at the start of the merge-write phase, before all keydir entries have been updated to point to new locations, creating a window where reads would fail even without concurrent access
- `log-structured-compact-closes-active-file` — In `log-structured-hash-table/bitcask.py`, compaction closes `_active_file` during Phase 3, making all reads and writes to the active segment impossible until Phase 5 reopens it under a new name
- `neither-bitcask-has-atomic-switchover` — Neither implementation maintains the paper's invariant that old files are deleted only after the keydir is fully updated; both interleave keydir mutation with file handle closure and file deletion
- `crash-between-delete-and-rename-loses-data` — In both implementations, a crash after old segment files are deleted but before the active file is renamed leaves committed data permanently unrecoverable, because no manifest or log records the intended state transition
- `single-threaded-assumption-hides-ordering-bugs` — Both implementations document "no concurrent access" as a precondition, which masks the fact that even the sequential operation ordering violates the paper's atomicity requirements and creates crash-safety gaps

