# Proposed Beliefs

Edit each entry: change `[ACCEPT/REJECT]` to `[ACCEPT]` or `[REJECT]`.
Then run: `code-expert accept-beliefs`

---

**Generated:** 2026-05-28
**Source:** 5 entries from entries/
**Model:** claude

### [ACCEPT] btree-wal-before-data
Every page mutation is logged to the WAL (with fsync) before being written to the data file; recovery replays the WAL, so no committed write is lost on crash.
- Source: entries/2026/05/28/b-tree-storage-engine-btree.md

### [ACCEPT] btree-page-zero-is-metadata
Page 0 is always the metadata page (root_page, height, total_keys, next_free_page, free_list_head); the root page is never page 0.
- Source: entries/2026/05/28/b-tree-storage-engine-btree.md

### [ACCEPT] btree-no-underflow-rebalancing
Delete does not rebalance underfilled nodes; the only cleanup is freeing empty non-leftmost leaves at depth 2 — a deliberate simplification over production B-trees.
- Source: entries/2026/05/28/b-tree-storage-engine-btree.md

### [ACCEPT] btree-leaf-sibling-chain
Leaves store a `next_sibling` pointer forming a left-to-right chain terminated by `NO_SIBLING` (0xFFFFFFFF), enabling range scans without touching internal nodes.
- Source: entries/2026/05/28/b-tree-storage-engine-btree.md

### [ACCEPT] btree-recursive-insert-returns-union
`_insert` returns a discriminated union: `None` (updated existing), `'inserted'` (new key, no split), or `(mid_key, new_page)` (split happened); root splits are handled in `put()`, not inside the recursion.
- Source: entries/2026/05/28/b-tree-storage-engine-btree.md

### [ACCEPT] btree-free-list-intrusive
Freed pages store the "next free" pointer in their own body at `HEADER_SIZE` offset, forming an intrusive singly-linked list with no separate allocator data structure.
- Source: entries/2026/05/28/b-tree-storage-engine-btree.md

### [ACCEPT] btree-stdlib-only
The B-tree module uses only stdlib imports (os, struct, zlib, bisect) with zero external dependencies.
- Source: entries/2026/05/28/b-tree-storage-engine-btree.md

### [ACCEPT] btree-counters-reset-per-operation
`pm.reset_counters()` is called at the start of each public method, so `stats.pages_read/written` reflects only the most recent operation.
- Source: entries/2026/05/28/b-tree-storage-engine-btree.md

### [ACCEPT] bitcask-tombstone-is-empty-string
Deletion is represented by writing a record with an empty-string value; both `_scan_data_file` (checks `val_size == 0`) and `get` (checks `value == ""`) use this convention.
- Source: entries/2026/05/28/hash-index-storage-bitcask.md

### [ACCEPT] bitcask-single-active-writer
Exactly one data file is open for writes at any time; all other files are immutable and read-only.
- Source: entries/2026/05/28/hash-index-storage-bitcask.md

### [ACCEPT] bitcask-hint-files-skip-scan
When a `.hint` file exists for a file ID, `_rebuild_index` loads it instead of scanning the data file; hint files have no checksum validation, so they must be written atomically with their data files during compaction.
- Source: entries/2026/05/28/hash-index-storage-bitcask.md

### [ACCEPT] bitcask-no-checksum-validation
Records have no CRC or integrity checksum; the only corruption guard is the `assert read_key == key` in `get()`, which catches index/data mismatches but not bit-rot.
- Source: entries/2026/05/28/hash-index-storage-bitcask.md

### [ACCEPT] bitcask-compaction-renumbers-active
After compaction, merged files get IDs above the old active file, and the active file is renamed to an even higher ID to maintain the monotonically increasing file ID invariant.
- Source: entries/2026/05/28/hash-index-storage-bitcask.md

### [ACCEPT] bitcask-keydir-is-sole-index
The in-memory `keydir` dict is the only index; every live key has exactly one `KeyEntry` pointing to its most recent non-tombstone record on disk.
- Source: entries/2026/05/28/hash-index-storage-bitcask.md

### [ACCEPT] bitcask-fsync-per-record-default
`sync_writes` defaults to `True`, meaning every `_write_record` call triggers an `fsync` — durable by default at significant write throughput cost.
- Source: entries/2026/05/28/hash-index-storage-bitcask.md

### [ACCEPT] lsm-wal-before-memtable
Every `put()` and `delete()` writes to the WAL before inserting into the memtable, ensuring no acknowledged write is lost on crash.
- Source: entries/2026/05/28/log-structured-merge-tree-lsm.md

### [ACCEPT] lsm-tombstone-is-empty-bytes
Deletion is represented as `TOMBSTONE = b""` (empty bytes), which means empty-byte-string values and deleted keys are indistinguishable at the storage layer.
- Source: entries/2026/05/28/log-structured-merge-tree-lsm.md

### [ACCEPT] lsm-compaction-is-full-merge
`compact()` merges all SSTables into a single new SSTable (size-tiered, single-level), not incremental or leveled — simpler but with higher space amplification.
- Source: entries/2026/05/28/log-structured-merge-tree-lsm.md

### [ACCEPT] lsm-heapq-imported-unused
`heapq` is imported but never referenced in the code; `compact()` uses list sorting (`sort by (key, -seq)`) instead of a heap-based k-way merge.
- Source: entries/2026/05/28/log-structured-merge-tree-lsm.md

### [ACCEPT] lsm-sparse-index-default-16
The SSTable sparse index samples every 16th key by default; a point lookup binary-searches the sparse index then linear-scans up to 16 entries within the candidate block.
- Source: entries/2026/05/28/log-structured-merge-tree-lsm.md

### [ACCEPT] lsm-uses-sortedcontainers
The LSM memtable uses `sortedcontainers.SortedDict` — the only external dependency across the four storage engine modules examined so far.
- Source: entries/2026/05/28/log-structured-merge-tree-lsm.md

### [ACCEPT] lsm-wal-truncated-after-sstable-write
The WAL is truncated only after a successful SSTable write; if a crash occurs between SSTable.write and WAL.truncate, replay harmlessly re-inserts into the memtable (idempotent because it's a dict).
- Source: entries/2026/05/28/log-structured-merge-tree-lsm.md

### [ACCEPT] lsm-newest-first-read-path
`get()` searches memtable, then immutable memtables, then SSTables in reverse-seq order (newest first), returning the first match; this guarantees newer writes shadow older ones without scanning all levels.
- Source: entries/2026/05/28/log-structured-merge-tree-lsm.md

### [ACCEPT] ddia-modules-self-contained
Each top-level directory is a self-contained module with no cross-module imports; implementations are intentionally standalone so each concept can be understood in isolation.
- Source: entries/2026/05/28/scan-ddia-implementations.md

### [ACCEPT] ddia-three-file-pattern
Most modules follow a consistent three-file pattern: one implementation file, one test file, and a `tester_test_*.py` file (likely a meta-test or test harness validator).
- Source: entries/2026/05/28/scan-ddia-implementations.md

### [ACCEPT] ddia-pure-python-stdlib
All implementations are Python; the storage engine modules use only stdlib except for `sortedcontainers` in the LSM tree — no frameworks or production infrastructure.
- Source: entries/2026/05/28/scan-ddia-implementations.md

### [ACCEPT] wal-replay-ignores-commit
`replay()` returns all PUT/DELETE records regardless of whether they are followed by a COMMIT record, meaning uncommitted batches are replayed identically to committed ones.
- Source: entries/2026/05/28/write-ahead-log-wal.md

### [ACCEPT] wal-crc-does-not-cover-seqnum
The CRC32 checksum covers only `op_type + key + value`; a corrupted sequence number would not be detected by integrity checking.
- Source: entries/2026/05/28/write-ahead-log-wal.md

### [ACCEPT] wal-truncate-not-crash-safe
`truncate()` rewrites WAL files in place without atomic rename, so a crash during truncation can leave the log in an inconsistent state.
- Source: entries/2026/05/28/write-ahead-log-wal.md

### [ACCEPT] wal-crc-mismatch-halts-all-replay
`_read_all_records` catches CRC `ValueError` and stops iteration entirely (returns, not continues), so corruption in any file aborts reading of all subsequent files too.
- Source: entries/2026/05/28/write-ahead-log-wal.md

### [ACCEPT] wal-batch-always-fsyncs
`append_batch()` always force-fsyncs regardless of the configured sync mode, because batch atomicity requires the COMMIT marker to be durable before returning.
- Source: entries/2026/05/28/write-ahead-log-wal.md

### [ACCEPT] wal-seq-nums-strictly-monotonic
Sequence numbers increment under a threading.Lock and are never reused; on recovery, all WAL files are scanned to find the high-water mark so new records continue the sequence.
- Source: entries/2026/05/28/write-ahead-log-wal.md

### [ACCEPT] wal-partial-read-is-eof
`_read_record` returns `None` on partial/short reads (torn writes), treated as EOF; this is distinct from CRC mismatch which raises `ValueError` — the distinction separates "crash during write" from "data corruption."
- Source: entries/2026/05/28/write-ahead-log-wal.md



---

**Generated:** 2026-05-28
**Source:** 52 entries from entries/
**Model:** claude

### [REJECT] btree-delete-no-rebalance
Duplicates existing belief `btree-no-underflow-rebalancing`
- Source: entries/2026/05/28/b-tree-storage-engine-btree-_delete.md

### [ACCEPT] btree-delete-cleanup-depth2-only
Empty leaf cleanup only triggers when the current internal node is at depth 2 (direct parent of leaves) and the empty child is not the leftmost; empty leaves at depth > 2 or at index 0 are silently left in place
- Source: entries/2026/05/28/b-tree-storage-engine-btree-_delete.md

### [ACCEPT] btree-free-page-bypasses-wal
`pm.free_page()` called during deletion writes directly to the data file without WAL logging, creating a crash-consistency gap if the process fails between the free and the WAL commit
- Source: entries/2026/05/28/b-tree-storage-engine-btree-_delete.md

### [ACCEPT] btree-delete-three-valued-return
`_delete` returns `False` (not found), `True` (deleted, no cleanup needed), or the string `'empty'` (leaf now empty, parent should attempt cleanup); the public `delete()` collapses this to `bool`
- Source: entries/2026/05/28/b-tree-storage-engine-btree-_delete.md

### [ACCEPT] btree-leaf-sibling-patched-on-remove
When an empty leaf is removed during deletion, `_delete` patches the previous sibling's `next_sibling` pointer to splice the empty leaf out of the linked list used by `range_scan` and `__iter__`
- Source: entries/2026/05/28/b-tree-storage-engine-btree-_delete.md

### [REJECT] btree-insert-three-way-return
Duplicates existing belief `btree-recursive-insert-returns-union`
- Source: entries/2026/05/28/b-tree-storage-engine-btree-_insert.md

### [ACCEPT] btree-leaf-split-copies-key
Leaf splits copy the midpoint key into both the parent and the right leaf (key remains searchable in the leaf), while internal splits promote the midpoint key out of both children into the parent only
- Source: entries/2026/05/28/b-tree-storage-engine-btree-_insert.md

### [ACCEPT] btree-insert-no-wal-commit
`_insert` never calls `wal.commit()`; the caller (`put`) is solely responsible for committing after all page writes and metadata updates complete, making the entire `put` atomic with respect to crash recovery
- Source: entries/2026/05/28/b-tree-storage-engine-btree-_insert.md

### [ACCEPT] btree-allocate-page-outside-wal
Page allocation during splits modifies metadata through `PageManager.write_meta` (not WAL-logged), creating a potential crash-safety gap if the process fails between allocation and the subsequent WAL commit
- Source: entries/2026/05/28/b-tree-storage-engine-btree-_insert.md

### [REJECT] btree-sibling-chain-maintained
Duplicates existing belief `btree-leaf-sibling-chain`
- Source: entries/2026/05/28/b-tree-storage-engine-btree-_insert.md

### [ACCEPT] leaf-wire-format-is-header-sibling-entries
Leaf pages are laid out as `[type:1B][num_keys:2B][next_sibling:4B][entries...]` where each entry is `[key_len:2B][key][val_len:2B][val]`, all big-endian
- Source: entries/2026/05/28/b-tree-storage-engine-btree-_serialize_leaf.md

### [ACCEPT] serialize-leaf-is-pure
`_serialize_leaf` is a pure function with no side effects; all I/O happens in the caller via `_wal_write_page`
- Source: entries/2026/05/28/b-tree-storage-engine-btree-_serialize_leaf.md

### [REJECT] leaf-sibling-forms-linked-list
Duplicates existing belief `btree-leaf-sibling-chain`
- Source: entries/2026/05/28/b-tree-storage-engine-btree-_serialize_leaf.md

### [ACCEPT] zip-truncation-risk
If `keys` and `values` lists passed to `_serialize_leaf` have mismatched lengths, `zip` silently truncates to the shorter list while `num_keys` in the header reflects the longer, producing a corrupt page
- Source: entries/2026/05/28/b-tree-storage-engine-btree-_serialize_leaf.md

### [ACCEPT] values-must-be-bytes
B-tree values are written as raw bytes with no encoding; keys accept `str` (auto-encoded to UTF-8) or `bytes`, but values must already be `bytes`
- Source: entries/2026/05/28/b-tree-storage-engine-btree-_serialize_leaf.md

### [ACCEPT] allocate-page-prefers-free-list
`allocate_page` always recycles from the free list before extending the file with a new page, keeping the data file compact after deletions
- Source: entries/2026/05/28/b-tree-storage-engine-btree-allocate_page.md

### [REJECT] allocate-page-not-wal-protected
Duplicates `btree-allocate-page-outside-wal` (same claim about metadata writes bypassing WAL)
- Source: entries/2026/05/28/b-tree-storage-engine-btree-allocate_page.md

### [REJECT] free-list-is-singly-linked
Duplicates existing belief `btree-free-list-intrusive` (free list threaded through freed pages)
- Source: entries/2026/05/28/b-tree-storage-engine-btree-allocate_page.md

### [ACCEPT] allocate-page-no-initialization
`allocate_page` returns a page number containing whatever stale data was previously on disk; the caller must overwrite it with valid page content before the next WAL commit
- Source: entries/2026/05/28/b-tree-storage-engine-btree-allocate_page.md

### [ACCEPT] free-list-layout-coupled
`allocate_page` and `free_page` share an implicit contract on free-list node layout (3-byte zeroed header + 4-byte big-endian next-pointer); neither validates the other's output, so a format change in one silently breaks the other
- Source: entries/2026/05/28/b-tree-storage-engine-btree-allocate_page.md

### [ACCEPT] wal-commit-sync-before-truncate
`WAL.commit` always fsyncs the data file before truncating the WAL file; reversing this order would create a crash-safety hole where committed data could be lost
- Source: entries/2026/05/28/b-tree-storage-engine-btree-commit.md

### [ACCEPT] wal-commit-clears-all-entries
`WAL.commit` clears the entire WAL unconditionally via truncate; there is no partial commit or transaction grouping within a single WAL file
- Source: entries/2026/05/28/b-tree-storage-engine-btree-commit.md

### [ACCEPT] wal-seq-reset-on-commit
The WAL sequence counter resets to 0 on every commit; sequence numbers are only meaningful within a single uncommitted transaction window
- Source: entries/2026/05/28/b-tree-storage-engine-btree-commit.md

### [ACCEPT] wal-is-redo-only
The B-tree WAL uses redo-only recovery (replay logged page images forward); there is no undo log, so incomplete pre-commit writes in the data file are overwritten by WAL replay to restore consistency
- Source: entries/2026/05/28/b-tree-storage-engine-btree-commit.md


### [ACCEPT] range-scan-half-open-interval
`BTree.range_scan(start_key, end_key)` returns keys in the half-open interval `[start_key, end_key)` — start-inclusive, end-exclusive; `end_key=None` scans to the last key.
- Source: entries/2026/05/28/b-tree-storage-engine-btree-range_scan.md

### [ACCEPT] range-scan-materializes-results
`range_scan` collects all matching `(key, value)` pairs into a list in memory rather than yielding lazily; large ranges consume proportional memory.
- Source: entries/2026/05/28/b-tree-storage-engine-btree-range_scan.md

### [ACCEPT] range-scan-follows-sibling-chain
`range_scan` walks the leaf-level linked list via `next_sibling` pointers rather than re-descending the tree, giving O(height + leaf pages in range) page reads.
- Source: entries/2026/05/28/b-tree-storage-engine-btree-range_scan.md

### [ACCEPT] range-scan-no-cycle-guard
`range_scan` has no visited-set or cycle detection on the leaf sibling chain; a corrupted `next_sibling` pointer would cause an infinite loop.
- Source: entries/2026/05/28/b-tree-storage-engine-btree-range_scan.md

### [REJECT] range-scan-resets-io-counters
Duplicate of existing belief `btree-counters-reset-per-operation`.
- Source: entries/2026/05/28/b-tree-storage-engine-btree-range_scan.md

### [ACCEPT] btree-wal-no-data-fsync
`PageManager.write_page` calls `flush()` but never `os.fsync()`, so written pages may not be durable on crash even after the WAL has committed.
- Source: entries/2026/05/28/b-tree-storage-engine-fix-plan.md

### [ACCEPT] btree-wal-truncate-before-data-sync
`WAL.commit()` truncates the WAL before the data file is fsynced, creating a crash window where both the log and the data can be lost.
- Source: entries/2026/05/28/b-tree-storage-engine-fix-plan.md

### [ACCEPT] btree-metadata-bypasses-wal
`BTree._write_meta` writes root pointer and free list head directly to the data file without logging through the WAL, making metadata updates not crash-safe.
- Source: entries/2026/05/28/b-tree-storage-engine-fix-plan.md

### [ACCEPT] btree-delete-leaks-pages
`BTree._delete` removes keys from leaves but never calls `free_page` on empty leaves, causing unbounded page file growth despite `PageManager` having a free list mechanism.
- Source: entries/2026/05/28/b-tree-storage-engine-fix-plan.md

### [REJECT] btree-fix-plan-execution-order
This describes dependency ordering within a planning document, not a code invariant — it will become irrelevant once the fixes land.
- Source: entries/2026/05/28/b-tree-storage-engine-fix-plan.md

### [ACCEPT] btree-max-keys-forces-splits
With `max_keys_per_page=4`, inserting a 5th key into a leaf triggers a page split; tests use this small fanout to exercise multi-level tree structure with few keys.
- Source: entries/2026/05/28/b-tree-storage-engine-test_btree.md

### [ACCEPT] btree-wal-replays-without-commit
The B-tree's WAL replays all valid (CRC-passing) entries on recovery regardless of whether a commit marker is present — uncommitted entries are applied without distinction.
- Source: entries/2026/05/28/b-tree-storage-engine-test_btree.md

### [ACCEPT] btree-crc32-skips-corrupt-entries
Corrupted WAL entries (CRC mismatch) are silently skipped during B-tree recovery rather than raising an error or halting replay.
- Source: entries/2026/05/28/b-tree-storage-engine-test_btree.md

### [ACCEPT] btree-lookup-reads-bounded-by-height
A single `BTree.get()` reads at most `height + 1` disk pages, verified via `pm.pages_read` after counter reset.
- Source: entries/2026/05/28/b-tree-storage-engine-test_btree.md

### [ACCEPT] btree-put-is-upsert
`BTree.put()` on an existing key updates the value in-place without incrementing `len(tree)` — it is an upsert, not a blind insert.
- Source: entries/2026/05/28/b-tree-storage-engine-tester_test_btree.md

### [ACCEPT] btree-survives-close-reopen
Data written before `BTree.close()` is fully recoverable by constructing a new `BTree` on the same directory, including updates and deletes.
- Source: entries/2026/05/28/b-tree-storage-engine-tester_test_btree.md

### [REJECT] btree-get-reads-at-most-height-plus-one-pages
Duplicate of `btree-lookup-reads-bounded-by-height` proposed from the sibling test entry.
- Source: entries/2026/05/28/b-tree-storage-engine-tester_test_btree.md

### [REJECT] btree-range-scan-excludes-end
Duplicate of `range-scan-half-open-interval` proposed from the range_scan entry.
- Source: entries/2026/05/28/b-tree-storage-engine-tester_test_btree.md

### [ACCEPT] bitcask-rebuild-sorts-ascending
`_rebuild_index` sorts file IDs ascending before scanning so that newer records overwrite older ones in the keydir, enforcing last-write-wins semantics.
- Source: entries/2026/05/28/hash-index-storage-bitcask-_rebuild_index.md

### [ACCEPT] bitcask-tombstone-handling-asymmetry
`_scan_data_file` removes tombstoned keys from `keydir` during rebuild, but `_load_hint_file` unconditionally inserts entries — it relies on compaction having already stripped tombstones before writing the hint file.
- Source: entries/2026/05/28/hash-index-storage-bitcask-_rebuild_index.md

### [REJECT] hint-file-is-fast-path
Duplicate of existing belief `bitcask-hint-files-skip-scan`.
- Source: entries/2026/05/28/hash-index-storage-bitcask-_rebuild_index.md

### [REJECT] no-corruption-detection-on-rebuild
Duplicate of existing belief `bitcask-no-checksum-validation`.
- Source: entries/2026/05/28/hash-index-storage-bitcask-_rebuild_index.md


### [ACCEPT] hint-file-no-fsync
Hint files are written without `fsync`; they are a best-effort startup optimization that can be lost on crash without data loss, since the data file remains authoritative.
- Source: entries/2026/05/28/hash-index-storage-bitcask-_write_hint_file.md

### [ACCEPT] hint-format-symmetry
`_write_hint_file` and `_load_hint_file` must use identical binary layouts: `[HINT_FORMAT header (24 bytes)][key_len uint32 (4 bytes)][key_bytes (variable)]` per entry; any change to one without the other breaks backward compatibility.
- Source: entries/2026/05/28/hash-index-storage-bitcask-_write_hint_file.md

### [ACCEPT] hint-written-only-during-compaction
`_write_hint_file` is called exclusively from `compact()`, never during normal `put`/`delete` operations; hint files only exist for compacted (merged) data files.
- Source: entries/2026/05/28/hash-index-storage-bitcask-_write_hint_file.md

### [ACCEPT] hint-file-no-integrity-check
Neither the hint file writer nor reader performs checksum validation; a truncated or corrupted hint file causes a `struct.error` or `IndexError` on startup rather than a graceful fallback to data file scanning.
- Source: entries/2026/05/28/hash-index-storage-bitcask-_write_hint_file.md

### [ACCEPT] bitcask-compact-preserves-timestamps
Compaction rewrites records with their original timestamps (not the time of compaction), preserving timestamp-based ordering across merges.
- Source: entries/2026/05/28/hash-index-storage-bitcask-compact.md

### [ACCEPT] bitcask-compact-skips-active-file-keys
During compaction, keys whose `keydir` entry points to the active file are excluded from merge output; their immutable-file copies are treated as stale because the active file holds a newer value.
- Source: entries/2026/05/28/hash-index-storage-bitcask-compact.md

### [ACCEPT] bitcask-compact-not-crash-safe
Hash-index-storage compaction is non-atomic with no error handling: a crash between deleting old files and completing the rewrite can leave the store in an unrecoverable state with no rollback mechanism.
- Source: entries/2026/05/28/hash-index-storage-bitcask-compact.md

### [REJECT] bitcask-compact-renumbers-active-file
Duplicates existing belief `bitcask-compaction-renumbers-active`.
- Source: entries/2026/05/28/hash-index-storage-bitcask-compact.md

### [ACCEPT] bitcask-compact-writes-hint-files
Compaction produces `.hint` files alongside each merged data file (including mid-compaction when a merged file hits `max_file_size`), enabling O(keys) index rebuilds instead of O(records) full scans on next startup.
- Source: entries/2026/05/28/hash-index-storage-bitcask-compact.md

### [ACCEPT] bitcask-get-missing-returns-none
`BitcaskStore.get()` returns `None` for missing or deleted keys, never raises `KeyError`.
- Source: entries/2026/05/28/hash-index-storage-test_bitcask.md

### [ACCEPT] bitcask-delete-nonexistent-is-noop
Calling `delete()` on a key that doesn't exist in the store completes without error.
- Source: entries/2026/05/28/hash-index-storage-test_bitcask.md

### [ACCEPT] bitcask-compaction-preserves-observable-state
After `compact()`, every key returns the same value as before compaction and deleted keys remain absent; compaction changes only physical layout, never logical state.
- Source: entries/2026/05/28/hash-index-storage-test_bitcask.md

### [ACCEPT] bitcask-crash-recovery-without-hints
`BitcaskStore` can rebuild its in-memory index by scanning `.data` files alone when `.hint` files are missing, producing identical read results to a clean startup with hint files present.
- Source: entries/2026/05/28/hash-index-storage-test_bitcask.md

### [ACCEPT] bitcask-tests-disable-sync
All tests in `test_bitcask.py` pass `sync_writes=False`, meaning the durable fsync-per-write code path is untested.
- Source: entries/2026/05/28/hash-index-storage-test_bitcask.md

### [ACCEPT] scan-segment-ordering-invariant
`_scan_segment` must be called on segments in ascending segment-ID order for the index to correctly reflect last-write-wins semantics; no runtime check enforces this ordering — it is the caller's responsibility.
- Source: entries/2026/05/28/log-structured-hash-table-bitcask-_scan_segment.md

### [ACCEPT] corruption-terminates-scan
In `_scan_segment`, a CRC mismatch or short read stops scanning the entire segment; records after the corrupt point are silently lost from the index even if they are individually valid.
- Source: entries/2026/05/28/log-structured-hash-table-bitcask-_scan_segment.md

### [ACCEPT] tombstone-removes-cross-segment
A tombstone record in segment N removes the key from `_index` even if the key's live value was written in an earlier segment, because the index is a flat dict shared across all segment scans.
- Source: entries/2026/05/28/log-structured-hash-table-bitcask-_scan_segment.md

### [REJECT] scan-segment-is-read-only
Too trivial — a scan/recovery function being read-only with respect to disk is expected behavior, not a surprising invariant.
- Source: entries/2026/05/28/log-structured-hash-table-bitcask-_scan_segment.md

### [ACCEPT] utf8-decode-unguarded
If a segment contains a key with invalid UTF-8 bytes, `_scan_segment` raises an unhandled `UnicodeDecodeError` that aborts recovery for all subsequent segments.
- Source: entries/2026/05/28/log-structured-hash-table-bitcask-_scan_segment.md

### [ACCEPT] compact-skips-crc-validation
Log-structured-hash-table compaction reads records from frozen segments without verifying CRC checksums, unlike `_scan_segment` which stops at the first CRC mismatch; a corrupted record could be silently copied into the compacted segment.
- Source: entries/2026/05/28/log-structured-hash-table-bitcask-compact.md

### [REJECT] compact-preserves-active-highest-id
Substantially duplicates existing belief `bitcask-compaction-renumbers-active` — the invariant that the active segment has the highest ID is maintained by the renumbering mechanism that belief already describes.
- Source: entries/2026/05/28/log-structured-hash-table-bitcask-compact.md

### [ACCEPT] compact-not-crash-safe
Log-structured-hash-table compaction is non-atomic: the sequence of write-new, update-index, delete-old, rename-active has no transaction boundary, and a crash mid-compaction can leave orphaned segments or a misnamed active file.
- Source: entries/2026/05/28/log-structured-hash-table-bitcask-compact.md

### [ACCEPT] compact-filters-against-live-index
During log-structured-hash-table compaction, records in frozen segments are only kept if `_index` still points to that exact `(path, offset)`, correctly excluding keys that were overwritten in the active segment after the frozen segment was closed.
- Source: entries/2026/05/28/log-structured-hash-table-bitcask-compact.md

### [ACCEPT] compact-single-threaded-assumption
Log-structured-hash-table compaction has no synchronization protecting the index, file handles, or segment counter during its multi-step mutation sequence; concurrent access during compaction would corrupt state.
- Source: entries/2026/05/28/log-structured-hash-table-bitcask-compact.md


### [ACCEPT] bitcask-crc-covers-payload-only
CRC32 is computed over `key_bytes + value_bytes` only; the 12-byte header (which contains the CRC itself) is excluded from the checksum input
- Source: entries/2026/05/28/log-structured-hash-table-bitcask.md

### [ACCEPT] bitcask-get-opens-fresh-handle
`get()` opens a new file handle per call rather than using the `_file_handles` cache, making the handle cache effectively unused for reads
- Source: entries/2026/05/28/log-structured-hash-table-bitcask.md

### [ACCEPT] bitcask-recovery-order-is-ascending
Recovery scans segments in ascending ID order so that newer writes overwrite older index entries, and the highest-ID segment becomes the active segment for appending
- Source: entries/2026/05/28/log-structured-hash-table-bitcask.md

### [ACCEPT] bitcask-compaction-not-crash-safe
Compaction performs delete-then-rename without journaling; a crash mid-compaction can leave the store in a state that `_recover()` cannot reconstruct correctly
- Source: entries/2026/05/28/log-structured-hash-table-bitcask.md

### [ACCEPT] bitcask-hint-files-exclude-tombstones
`create_hint_files()` skips tombstone records and only emits entries whose index currently points to that segment+offset, so hint-based recovery cannot distinguish "key was deleted" from "key never existed in this segment"
- Source: entries/2026/05/28/log-structured-hash-table-bitcask.md

### [ACCEPT] bitcask-tombstone-semantics
`BitcaskStore.delete()` writes a tombstone record (`b"__BITCASK_TOMBSTONE__"`); `get()` returns `None` for tombstoned keys, and `compact()` removes both the original entry and the tombstone
- Source: entries/2026/05/28/log-structured-hash-table-test_bitcask.md

### [REJECT] bitcask-crc-per-record
Overlaps with `bitcask-crc-covers-payload-only` — the CRC-per-record framing and CorruptionError behavior are already captured there
- Source: entries/2026/05/28/log-structured-hash-table-test_bitcask.md

### [ACCEPT] bitcask-partial-write-safe
On startup recovery, the store skips incomplete records at segment tail (header present but payload truncated) without raising errors or losing previously committed data
- Source: entries/2026/05/28/log-structured-hash-table-test_bitcask.md

### [ACCEPT] bitcask-auto-compact-threshold
When the number of frozen segments exceeds `auto_compact_threshold` (default 5), compaction is triggered automatically during `put()` operations
- Source: entries/2026/05/28/log-structured-hash-table-test_bitcask.md

### [REJECT] bitcask-index-is-memory-only
Duplicates existing belief `bitcask-keydir-is-sole-index` — both describe the in-memory index as the sole authority for key lookups
- Source: entries/2026/05/28/log-structured-hash-table-test_bitcask.md

### [REJECT] compact-merges-all-sstables
Duplicates existing belief `lsm-compaction-is-full-merge`
- Source: entries/2026/05/28/log-structured-merge-tree-lsm-compact.md

### [ACCEPT] compact-newest-wins-by-seq
During compaction, the entry with the highest SSTable sequence number wins for each key; sequence numbers are per-SSTable (all entries in one SSTable share the same seq), not per-entry
- Source: entries/2026/05/28/log-structured-merge-tree-lsm-compact.md

### [ACCEPT] compact-purges-tombstones
Tombstones (`b""`) are permanently removed during compaction and never written to the output SSTable; deleted keys disappear entirely after merge
- Source: entries/2026/05/28/log-structured-merge-tree-lsm-compact.md

### [ACCEPT] compact-not-crash-safe
A crash between writing the new SSTable and deleting old ones leaves orphan files that `_load_existing_sstables` will reload (data-safe but wastes space); no journaling or atomic swap is used
- Source: entries/2026/05/28/log-structured-merge-tree-lsm-compact.md

### [ACCEPT] compact-no-concurrency-safety
`compact()` mutates `self._sstables` without locking; concurrent `_flush` or `get` calls during compaction can produce incorrect state or lose newly flushed SSTables
- Source: entries/2026/05/28/log-structured-merge-tree-lsm-compact.md

### [ACCEPT] range-scan-last-writer-wins
`range_scan` iterates sources oldest-to-newest (SSTables, then immutable memtables, then active memtable) so that dict overwrites naturally give each key the value from the newest source
- Source: entries/2026/05/28/log-structured-merge-tree-lsm-range_scan.md

### [ACCEPT] range-scan-half-open-interval
The scan range is `[start_key, end_key)`: inclusive on start, exclusive on end, enforced consistently by both `SSTable.scan` and `SortedDict.irange`
- Source: entries/2026/05/28/log-structured-merge-tree-lsm-range_scan.md

### [ACCEPT] range-scan-pure-read
`range_scan` has no side effects: it does not mutate tree state, trigger flushes, or modify the WAL
- Source: entries/2026/05/28/log-structured-merge-tree-lsm-range_scan.md

### [ACCEPT] range-scan-priority-field-unused
The priority integer stored in the `merged` dict during `range_scan` is never read or compared; correctness relies entirely on the source iteration order and dict overwrite semantics, making the priority field dead weight
- Source: entries/2026/05/28/log-structured-merge-tree-lsm-range_scan.md

### [ACCEPT] range-scan-no-concurrency-protection
`range_scan` can observe inconsistent state if a flush or compaction runs concurrently, since there is no snapshot isolation or locking
- Source: entries/2026/05/28/log-structured-merge-tree-lsm-range_scan.md

### [REJECT] lsm-range-scan-half-open
Duplicates `range-scan-half-open-interval` proposed from the more detailed range_scan entry
- Source: entries/2026/05/28/log-structured-merge-tree-test_lsm.md

### [ACCEPT] lsm-memtable-threshold-triggers-flush
When the number of entries in the active memtable reaches `memtable_threshold`, the memtable is automatically frozen and flushed to a new SSTable on disk
- Source: entries/2026/05/28/log-structured-merge-tree-test_lsm.md

### [REJECT] lsm-compaction-merges-all-to-one
Duplicates existing belief `lsm-compaction-is-full-merge`
- Source: entries/2026/05/28/log-structured-merge-tree-test_lsm.md

### [ACCEPT] lsm-wal-replays-on-reopen
Constructing a new `LSMTree` on an existing directory replays the WAL to recover unflushed memtable state; this is validated by the crash recovery test which skips `close()` and reopens
- Source: entries/2026/05/28/log-structured-merge-tree-test_lsm.md

### [REJECT] lsm-newest-write-wins-across-sstables
Covered by existing belief `lsm-newest-first-read-path` (for point lookups) combined with proposed `range-scan-last-writer-wins` (for range scans)
- Source: entries/2026/05/28/log-structured-merge-tree-test_lsm.md


### [ACCEPT] sstable-merge-dedup-newest-timestamp
When multiple SSTables contain the same key, `merge_sstables` keeps only the entry with the highest timestamp via heap ordering `(key, -timestamp)`, silently discarding all older versions.
- Source: entries/2026/05/28/sstable-and-compaction-sstable-merge_sstables.md

### [ACCEPT] merge-preserves-tombstones-by-default
`merge_sstables` writes tombstones (`value=None`) to the output unless `remove_tombstones=True` is explicitly passed, and no current call site in `CompactionManager` enables removal.
- Source: entries/2026/05/28/sstable-and-compaction-sstable-merge_sstables.md

### [ACCEPT] heap-tiebreak-uses-source-index
The merge heap tuple includes a `source_index` field solely to prevent Python from comparing `SSTableEntry` objects when `(key, -timestamp)` ties, which would raise `TypeError`.
- Source: entries/2026/05/28/sstable-and-compaction-sstable-merge_sstables.md

### [ACCEPT] merge-no-atomic-write
`merge_sstables` writes directly to `output_path` with no temp-file-and-rename pattern, so a crash mid-merge leaves a corrupt partial file at the target path.
- Source: entries/2026/05/28/sstable-and-compaction-sstable-merge_sstables.md

### [ACCEPT] merge-does-not-delete-inputs
`merge_sstables` never deletes or modifies input SSTable files; cleanup is the caller's responsibility, and the current `CompactionManager` callers also do not delete old files from disk.
- Source: entries/2026/05/28/sstable-and-compaction-sstable-merge_sstables.md

### [ACCEPT] sstable-tombstone-encoding-0xff
Tombstones are encoded on disk as a single `0xFF` byte in the value position; this is unambiguous because valid value-length fields are 4 bytes big-endian and cannot start with `0xFF` at realistic value sizes.
- Source: entries/2026/05/28/sstable-and-compaction-sstable.md

### [ACCEPT] sstable-writer-append-then-patch
`SSTableWriter` writes a placeholder header, appends all entries and the sparse index, then seeks back to byte 0 to patch the final entry count — an unfinished SSTable has `entry_count=0` and appears empty.
- Source: entries/2026/05/28/sstable-and-compaction-sstable.md

### [ACCEPT] sstable-sparse-index-every-nth
`SSTableReader` loads a sparse index into memory that records every `block_size`-th key and byte offset; point lookups binary-search the index then linearly scan within the block.
- Source: entries/2026/05/28/sstable-and-compaction-sstable.md

### [ACCEPT] sstable-range-scan-no-index-skip
`SSTableReader.range_scan(start, end)` scans from the beginning of the file rather than using the sparse index to skip ahead, making it O(N) regardless of range position.
- Source: entries/2026/05/28/sstable-and-compaction-sstable.md

### [ACCEPT] sstable-no-checksums
The SSTable format has no CRC or checksum on entries or blocks; the only validation is a magic-byte assertion (`MAGIC == b"SSTB"`) on file open.
- Source: entries/2026/05/28/sstable-and-compaction-sstable.md

### [ACCEPT] sstable-sorted-order-caller-responsibility
`SSTableWriter.add()` does not enforce sorted key order; violation silently corrupts binary search in `SSTableReader.get()`.
- Source: entries/2026/05/28/sstable-and-compaction-sstable.md

### [ACCEPT] sstable-tombstone-is-null-value
Tombstones are represented at the API level as `SSTableEntry` objects with `value=None`, not as a separate deletion record type.
- Source: entries/2026/05/28/sstable-and-compaction-test_sstable.md

### [ACCEPT] sstable-range-scan-end-exclusive
`range_scan(start, end)` is inclusive on `start` and exclusive on `end`.
- Source: entries/2026/05/28/sstable-and-compaction-test_sstable.md

### [ACCEPT] compaction-manager-supports-two-strategies
`CompactionManager` implements both `'size_tiered'` and `'leveled'` compaction strategies, selected by a string parameter.
- Source: entries/2026/05/28/sstable-and-compaction-test_sstable.md

### [ACCEPT] leveled-compaction-promotes-to-l1
After leveled compaction runs on L0 SSTables, the output readers have `level == 1` set by the caller.
- Source: entries/2026/05/28/sstable-and-compaction-test_sstable.md

### [ACCEPT] sstable-test-is-script-not-framework
`test_sstable.py` is a linear script using bare `assert` statements and `print` progress markers, not a pytest/unittest suite — it runs top-to-bottom in a `tempfile.TemporaryDirectory` context.
- Source: entries/2026/05/28/sstable-and-compaction-test_sstable.md

### [REJECT] delete-never-rebalances
Duplicates existing belief `btree-no-underflow-rebalancing`.
- Source: entries/2026/05/28/topic-b-tree-underflow-rebalancing.md

### [ACCEPT] btree-sibling-pointers-unused-for-merge
Leaf pages serialize a `next_sibling` pointer (via `_serialize_leaf` and `NO_SIBLING` sentinel) providing infrastructure for merge/redistribute, but no code path reads sibling pointers during deletion.
- Source: entries/2026/05/28/topic-b-tree-underflow-rebalancing.md

### [ACCEPT] btree-delete-skips-free-page
The delete path does not call `free_page` even when a leaf becomes completely empty; `PageManager`'s free list is used only by `allocate_page` after explicit `free_page` calls from other paths.
- Source: entries/2026/05/28/topic-b-tree-underflow-rebalancing.md

### [ACCEPT] btree-height-never-shrinks
Since delete never merges internal nodes or demotes the root, tree height only increases (on splits) and never decreases, regardless of how many keys are deleted.
- Source: entries/2026/05/28/topic-b-tree-underflow-rebalancing.md

### [ACCEPT] bitcask-compaction-not-crash-safe
Both Bitcask implementations delete old segment files before renaming the active segment, creating a crash window where committed data can be permanently lost.
- Source: entries/2026/05/28/topic-bitcask-compaction-atomicity.md

### [ACCEPT] no-compaction-manifest
None of the three storage engines (hash-index, log-structured-hash-table, LSM) use a manifest or compaction log to make segment replacement atomic; segment discovery is purely filesystem-based via directory listing.
- Source: entries/2026/05/28/topic-bitcask-compaction-atomicity.md

### [ACCEPT] lsm-compaction-duplicates-safe
A crash during LSM compaction produces duplicate entries (old and merged SSTables both present) rather than data loss, because newer SSTables take read precedence.
- Source: entries/2026/05/28/topic-bitcask-compaction-atomicity.md

### [ACCEPT] crc-no-cross-file-atomicity
The CRC32 integrity checks in `log-structured-hash-table/bitcask.py` detect corrupt records within a segment but provide no protection against the cross-file atomicity problem during compaction.
- Source: entries/2026/05/28/topic-bitcask-compaction-atomicity.md

### [ACCEPT] hint-file-orphan-crash-risk
In `hash-index-storage/bitcask.py`, a crash between deleting a data file and its hint file leaves an orphaned hint that directs reads to a nonexistent data file, causing hard failures on `get()`.
- Source: entries/2026/05/28/topic-bitcask-compaction-atomicity.md


## Entry: `topic-bitcask-crash-recovery.md`

### [ACCEPT] no-compaction-manifest
None of the three storage engines (hash-index-storage, log-structured-hash-table, log-structured-merge-tree) use a manifest file to track which data files are current; file discovery relies entirely on `os.listdir` directory scanning
- Source: entries/2026/05/28/topic-bitcask-crash-recovery.md

### [ACCEPT] delete-before-rename-ordering
Both Bitcask implementations delete old segment files (`os.remove`) before renaming the compacted replacement, creating a crash window where neither old nor properly-named new data is on disk
- Source: entries/2026/05/28/topic-bitcask-crash-recovery.md

### [ACCEPT] hint-files-are-optimization-only
Hint files accelerate index rebuild during recovery but carry no crash-consistency guarantees; they are not consulted to determine which data files constitute the valid current state
- Source: entries/2026/05/28/topic-bitcask-crash-recovery.md

### [ACCEPT] lsm-compaction-deletes-without-fsync-barrier
The LSM tree's `compact()` calls `os.remove(sst.path)` without an explicit `fsync` on the new SSTable or its parent directory beforehand, meaning the new file's data may not be durable when the old file is unlinked
- Source: entries/2026/05/28/topic-bitcask-crash-recovery.md

### [ACCEPT] index-is-volatile
All three storage engines rebuild their in-memory index (`keydir`, `_index`) entirely from on-disk files at startup; inconsistent file state after a crash produces a silently incomplete index with no error
- Source: entries/2026/05/28/topic-bitcask-crash-recovery.md

---

## Entry: `topic-bloom-filter-integration.md`

### [ACCEPT] bloom-filter-not-integrated
Neither LSM tree implementation (`lsm.py` or `sstable.py`) references or uses the bloom filter module; they exist as independent DDIA concept demonstrations with zero cross-module imports
- Source: entries/2026/05/28/topic-bloom-filter-integration.md

### [ACCEPT] bloom-double-hashing
`_hashes()` uses Kirschner-Mitzenmacher double hashing from a single MD5 digest, producing k positions as `(h1 + i*h2) % m` without k independent hash functions
- Source: entries/2026/05/28/topic-bloom-filter-integration.md

### [ACCEPT] bloom-serialization-footer-ready
`BloomFilter.to_bytes` packs a 12-byte header (`m`, `k`, `count` as three little-endian uint32s) followed by the raw bit array, matching the pattern used by SSTable footers for sparse indices
- Source: entries/2026/05/28/topic-bloom-filter-integration.md

### [ACCEPT] counting-bloom-supports-deletion
`CountingBloomFilter` uses saturating 4-bit counters so entries can be removed via `remove()`, which standard `BloomFilter` cannot do — relevant for maintaining filters over mutable data structures
- Source: entries/2026/05/28/topic-bloom-filter-integration.md

### [ACCEPT] scalable-bloom-tightens-fpr
Each new slice in `ScalableBloomFilter` uses FPR of `p * (ratio ^ slice_index)`, so the aggregate false positive rate stays bounded even as the filter grows unboundedly
- Source: entries/2026/05/28/topic-bloom-filter-integration.md

---

## Entry: `topic-btree-split-and-merge-strategy.md`

### [REJECT] btree-metadata-tracks-height-and-total-keys
Overlaps with existing belief `btree-page-zero-is-metadata` which already establishes that page 0 is the metadata page
- Source: entries/2026/05/28/topic-btree-split-and-merge-strategy.md

### [ACCEPT] btree-delete-is-immediate-not-tombstone
Deletes decrement `total_keys` immediately and the key becomes unreadable at once; no tombstone entries are written and no deferred compaction pass is needed
- Source: entries/2026/05/28/topic-btree-split-and-merge-strategy.md

### [ACCEPT] btree-empty-leaf-freed-on-delete
When all keys are removed from a non-root leaf, the page is returned to the free list via `PageManager.free_page()` rather than persisting as an empty node
- Source: entries/2026/05/28/topic-btree-split-and-merge-strategy.md

### [ACCEPT] btree-wal-provides-split-atomicity
Multi-page operations (splits, deletes that free pages) are made crash-safe by writing all page modifications to the WAL before applying them to the data file
- Source: entries/2026/05/28/topic-btree-split-and-merge-strategy.md

### [REJECT] btree-leaves-form-sibling-linked-list
Duplicates existing belief `btree-leaf-sibling-chain`
- Source: entries/2026/05/28/topic-btree-split-and-merge-strategy.md

---

## Entry: `topic-concurrent-btree-deletion.md`

### [REJECT] delete-has-no-merge
Duplicates existing belief `btree-no-underflow-rebalancing`
- Source: entries/2026/05/28/topic-concurrent-btree-deletion.md

### [ACCEPT] no-concurrency-primitives
The entire B-tree implementation contains zero locking, latching, mutex, or thread-safety constructs; it is strictly single-threaded with no concurrent-access support
- Source: entries/2026/05/28/topic-concurrent-btree-deletion.md

### [REJECT] free-list-reclaims-empty-leaves
Overlaps with proposed `btree-empty-leaf-freed-on-delete` and existing `btree-free-list-intrusive`; the reclamation trigger and the free list mechanism are already covered
- Source: entries/2026/05/28/topic-concurrent-btree-deletion.md

### [ACCEPT] sibling-field-is-structural-only
Leaf pages carry a `next_sibling` field in their serialized format (`NO_SIBLING = 0xFFFFFFFF` sentinel), but no tree operation uses sibling pointers for traversal or merging — it is scaffolding, not a live data structure. Note: may refine existing `btree-leaf-sibling-chain`.
- Source: entries/2026/05/28/topic-concurrent-btree-deletion.md

### [ACCEPT] wal-provides-crash-safety-not-concurrency
The WAL syncs with `os.fsync` for durability guarantees only; it has no role in coordinating concurrent access, consistent with the single-threaded design
- Source: entries/2026/05/28/topic-concurrent-btree-deletion.md

---

## Entry: `topic-counting-bloom-compaction-interaction.md`

### [ACCEPT] bloom-compaction-rebuild-not-patch
For SSTable compaction, rebuilding a fresh `BloomFilter` during the merge write pass is simpler and more correct than incrementally patching a `CountingBloomFilter` with `remove()`; compaction already pays O(N) I/O so filter construction adds only constant-factor overhead
- Source: entries/2026/05/28/topic-counting-bloom-compaction-interaction.md

### [ACCEPT] counting-bloom-remove-needs-drop-tracking
Using `CountingBloomFilter.remove()` during compaction would require `merge_sstables` to report which keys were discarded, which the current implementation does not do — duplicates and tombstones are silently skipped in the merge loop
- Source: entries/2026/05/28/topic-counting-bloom-compaction-interaction.md

### [REJECT] bloom-filter-not-integrated
Already proposed and accepted from the bloom-filter-integration entry above
- Source: entries/2026/05/28/topic-counting-bloom-compaction-interaction.md

### [ACCEPT] counting-bloom-useful-for-memtable
`CountingBloomFilter.remove()` is architecturally suited for maintaining filters over mutable in-memory structures (memtables) where keys can be overwritten without rescanning, not for SSTable compaction where the entire file is rewritten
- Source: entries/2026/05/28/topic-counting-bloom-compaction-interaction.md

### [ACCEPT] sstable-writer-is-natural-filter-builder
The `SSTableWriter.add()` → `finish()` lifecycle is the natural integration point for per-SSTable bloom filter construction, since keys are already iterated in sorted order during the write pass
- Source: entries/2026/05/28/topic-counting-bloom-compaction-interaction.md


### [ACCEPT] no-directory-fsync-anywhere
No implementation in the codebase calls `os.fsync()` on a directory file descriptor; all fsync calls target data file descriptors only
- Source: entries/2026/05/28/topic-directory-fsync-semantics.md

### [ACCEPT] rename-without-dir-barrier
Both Bitcask compaction paths (`hash-index-storage/bitcask.py:297`, `log-structured-merge-tree/bitcask.py:301`) perform `os.rename()` without a subsequent directory fsync, making the rename non-durable on ext4 and XFS
- Source: entries/2026/05/28/topic-directory-fsync-semantics.md

### [ACCEPT] apfs-masks-linux-fsync-bugs
macOS APFS provides implicit rename durability through its CoW transaction model, so the missing directory fsync is a latent bug that only surfaces on Linux filesystems (ext4, XFS)
- Source: entries/2026/05/28/topic-directory-fsync-semantics.md

### [ACCEPT] lsm-wal-has-no-fsync
The WAL class in `log-structured-merge-tree/lsm.py` calls `flush()` but never `os.fsync()`, making it strictly weaker than the standalone WAL which offers configurable sync modes
- Source: entries/2026/05/28/topic-directory-fsync-semantics.md

### [ACCEPT] lsm-no-synchronization
`LSMTree` in `lsm.py` has zero locking, atomic swaps, or synchronization primitives; all shared state (`_memtable`, `_sstables`) is mutated in-place without protection
- Source: entries/2026/05/28/topic-lsm-concurrency-safety.md

### [ACCEPT] flush-clears-then-appends
`_flush()` both clears `self._memtable` and appends to `self._sstables`, creating a window where data exists in neither location if a concurrent reader checks between the two mutations
- Source: entries/2026/05/28/topic-lsm-concurrency-safety.md

### [ACCEPT] compact-deletes-old-sstable-files
`compact()` removes old SSTable files from disk after merging, meaning any concurrent reader holding a reference to a deleted SSTable reads stale data (on Unix) or crashes (on Windows)
- Source: entries/2026/05/28/topic-lsm-concurrency-safety.md

### [ACCEPT] range-scan-merges-memtable-and-sstables
`range_scan()` combines results from both the active memtable and all SSTables in `self._sstables`, making it sensitive to mutations of either data structure during iteration
- Source: entries/2026/05/28/topic-lsm-concurrency-safety.md

### [ACCEPT] lsm-tests-single-threaded-only
The LSM test suite (`test_lsm.py`) runs all operations sequentially with no concurrency, so race conditions between `_flush()`, `compact()`, and `range_scan()` are never exercised
- Source: entries/2026/05/28/topic-lsm-concurrency-safety.md

### [ACCEPT] wal-record-length-prefixed
Every WAL record is prefixed with a 4-byte little-endian length covering all subsequent fields (CRC through value), enabling the reader to know exactly how many bytes to consume
- Source: entries/2026/05/28/topic-lsm-crash-recovery-wal-format.md

### [REJECT] wal-crc-covers-payload-only
Duplicates existing belief `wal-crc-does-not-cover-seqnum`
- Source: entries/2026/05/28/topic-lsm-crash-recovery-wal-format.md

### [REJECT] wal-corruption-stops-file-scan
Duplicates existing beliefs `wal-crc-mismatch-halts-all-replay` and `wal-partial-read-is-eof`
- Source: entries/2026/05/28/topic-lsm-crash-recovery-wal-format.md

### [ACCEPT] wal-batch-atomicity-via-commit-record
A batch write is only considered complete if its trailing COMMIT record is present and passes CRC; incomplete batches (missing commit) are discarded during replay
- Source: entries/2026/05/28/topic-lsm-crash-recovery-wal-format.md

### [ACCEPT] lsm-no-manifest
The LSM tree has no persistent metadata log (MANIFEST); the set of live SSTable files exists only in memory (`self._sstables`) and is not crash-recoverable independently of directory scanning
- Source: entries/2026/05/28/topic-manifest-based-compaction.md

### [ACCEPT] compaction-not-atomic
`LSMTree.compact()` performs a multi-step file swap (write new SSTable, update in-memory list, delete old files) with no mechanism to make the transition atomic across crashes
- Source: entries/2026/05/28/topic-manifest-based-compaction.md

### [ACCEPT] wal-protects-data-not-metadata
The WAL protects user key-value writes but does not record SSTable-level state transitions (flush, compaction, level assignment), leaving metadata changes unprotected across crashes
- Source: entries/2026/05/28/topic-manifest-based-compaction.md

### [ACCEPT] sstable-metadata-level-field-unused
`SSTableMetadata.level` in `sstable-and-compaction/sstable.py` exists as a field but has no persistent backing; level assignments would be lost on restart without a MANIFEST equivalent
- Source: entries/2026/05/28/topic-manifest-based-compaction.md

### [ACCEPT] lsm-range-scan-materializes-all
`lsm.py:range_scan` loads all matching entries from every SSTable and memtable into a dict before returning any results, making first-result latency proportional to total result set size
- Source: entries/2026/05/28/topic-merge-iterator-vs-dict-merge.md

### [REJECT] heapq-imported-but-unused-for-merge
Duplicates existing belief `lsm-heapq-imported-unused`
- Source: entries/2026/05/28/topic-merge-iterator-vs-dict-merge.md

### [ACCEPT] sstable-scan-is-streaming
`SSTable.scan()` and `scan_all()` use Python generators (yield), meaning individual SSTables already support streaming — only the cross-SSTable merge layer materializes
- Source: entries/2026/05/28/topic-merge-iterator-vs-dict-merge.md

### [ACCEPT] compaction-buffers-all-entries
`lsm.py:compact` collects all merged entries into an in-memory list before writing the output SSTable, requiring O(n) memory proportional to total data rather than O(k) proportional to number of input SSTables
- Source: entries/2026/05/28/topic-merge-iterator-vs-dict-merge.md

### [ACCEPT] range-scan-newer-wins-by-priority
The range scan merge uses an incrementing priority counter where higher priority = newer source; the dict naturally deduplicates by overwriting older entries for the same key
- Source: entries/2026/05/28/topic-merge-iterator-vs-dict-merge.md


### [ACCEPT] postgres-defers-btree-rebalancing-to-vacuum
PostgreSQL's `nbtree` never merges or redistributes siblings on delete; it marks tuples dead and relies on VACUUM to reclaim space, trading immediate space efficiency for throughput under concurrency
- Source: entries/2026/05/28/topic-postgres-nbtree-lazy-deletion.md

### [ACCEPT] btree-impl-asymmetric-split-vs-merge
The reference implementation fully propagates splits upward through `_insert`'s return value but swallows the `'empty'` signal in `_delete` after one level of cleanup, making insertion structurally complete but deletion structurally incomplete
- Source: entries/2026/05/28/topic-postgres-nbtree-lazy-deletion.md

### [ACCEPT] tree-height-monotonically-increases
Since neither the reference implementation nor PostgreSQL merges internal nodes on delete, tree height only grows (on root splits) and never shrinks, even under heavy deletion
- Source: entries/2026/05/28/topic-postgres-nbtree-lazy-deletion.md

### [REJECT] sibling-chain-enables-but-unused-for-rebalancing
Redundant — combines `btree-leaf-sibling-chain` (sibling pointers exist) and `btree-no-underflow-rebalancing` (no rebalancing happens) without adding a new testable claim
- Source: entries/2026/05/28/topic-postgres-nbtree-lazy-deletion.md

### [REJECT] deferred-cleanup-is-production-pattern
Design principle generalization, not a testable fact about this codebase; better suited to a DDIA reading note than a code belief
- Source: entries/2026/05/28/topic-postgres-nbtree-lazy-deletion.md

### [ACCEPT] tombstone-removal-requires-full-key-coverage
A tombstone for key K can only be safely removed during compaction if every SSTable that could contain an older entry for K is included in that compaction run; otherwise the deleted key can be resurrected from a surviving SSTable
- Source: entries/2026/05/28/topic-tombstone-gc-safety.md

### [ACCEPT] compaction-manager-never-removes-tombstones
The `CompactionManager` in `sstable-and-compaction/sstable.py` always calls `merge_sstables` with the default `remove_tombstones=False`, making it conservatively correct but causing unbounded tombstone accumulation
- Source: entries/2026/05/28/topic-tombstone-gc-safety.md

### [ACCEPT] lsm-compact-removes-tombstones-safely
The LSM Tree's `compact()` method removes tombstones because it performs a full merge of all SSTables, guaranteeing no surviving SSTable can contain a superseded live value
- Source: entries/2026/05/28/topic-tombstone-gc-safety.md

### [ACCEPT] distributed-tombstone-removal-needs-replication-convergence
In the multi-leader replication module, tombstones cannot be safely removed until all replicas have received the delete, adding a replication-convergence constraint beyond the local compaction-coverage constraint
- Source: entries/2026/05/28/topic-tombstone-gc-safety.md

### [ACCEPT] btree-wal-truncate-is-safe
The B-tree's WAL `commit()` only truncates after fsyncing the data file, making truncation idempotent — a crash mid-truncation simply replays the WAL entries again on recovery
- Source: entries/2026/05/28/topic-truncation-crash-safety.md

### [REJECT] atomic-rename-requires-same-filesystem
General OS/POSIX knowledge, not a fact about this codebase — no code in the repository implements or relies on atomic rename
- Source: entries/2026/05/28/topic-truncation-crash-safety.md

### [REJECT] crash-safe-truncation-three-step
Describes a proposed fix pattern, not the current codebase behavior; the code does not implement write-to-temp + fsync + rename
- Source: entries/2026/05/28/topic-truncation-crash-safety.md

### [REJECT] orphaned-tmp-means-original-intact
Describes behavior of a proposed cleanup protocol that does not exist in the codebase
- Source: entries/2026/05/28/topic-truncation-crash-safety.md

### [REJECT] single-write-not-atomic-across-pages
General filesystem/OS knowledge, not a testable claim about this codebase
- Source: entries/2026/05/28/topic-wal-atomicity-guarantees.md

### [ACCEPT] wal-batch-relies-on-practical-atomicity
`append_batch()` buffers all operations plus COMMIT into a single `write()` call to minimize the partial-write window, but the code acknowledges this is not a true atomicity guarantee — a crash during the write can persist a prefix without the COMMIT marker
- Source: entries/2026/05/28/topic-wal-atomicity-guarantees.md

### [REJECT] replay-does-not-enforce-commit-boundary
Duplicate of existing belief `wal-replay-ignores-commit`
- Source: entries/2026/05/28/topic-wal-atomicity-guarantees.md

### [ACCEPT] wal-uses-crc32-not-sha
WAL integrity uses `zlib.crc32` (32-bit, non-cryptographic); it detects accidental corruption but not intentional tampering
- Source: entries/2026/05/28/topic-wal-checksum-format.md

### [REJECT] crc32-covers-op-key-value-only
Duplicate of existing belief `wal-crc-does-not-cover-seqnum` (same fact stated from the opposite direction)
- Source: entries/2026/05/28/topic-wal-checksum-format.md

### [REJECT] wal-stops-at-first-corruption
Duplicate of existing belief `wal-crc-mismatch-halts-all-replay`
- Source: entries/2026/05/28/topic-wal-checksum-format.md

### [ACCEPT] record-format-is-length-prefixed
Each WAL record starts with a 4-byte little-endian length prefix (not counting itself), followed by CRC, seq_num, op_type, and length-prefixed key/value pairs — minimum 25 bytes per record for empty key/value
- Source: entries/2026/05/28/topic-wal-checksum-format.md

### [REJECT] corruption-test-exploits-tail-position
Test strategy detail, not an architectural invariant — describes how `test_corruption` works rather than a property the codebase must maintain
- Source: entries/2026/05/28/topic-wal-checksum-format.md


### [ACCEPT] wal-iterate-preserves-all-ops
`iterate()` yields every record including COMMIT and CHECKPOINT markers, providing the raw stream needed for commit-aware recovery logic built on top of `replay()`
- Source: entries/2026/05/28/topic-wal-commit-semantics.md

### [ACCEPT] wal-batch-single-write
`append_batch()` serializes all data records plus the trailing COMMIT marker into one `bytearray` and issues a single `fd.write()` call, relying on OS write atomicity for small batches
- Source: entries/2026/05/28/topic-wal-commit-semantics.md

### [ACCEPT] wal-replay-no-atomicity-check
`replay()` does not verify that batch operations have a corresponding COMMIT record; partial batches from a mid-write crash would be replayed as if committed, since replay filters only by record type, not by batch completeness
- Source: entries/2026/05/28/topic-wal-commit-semantics.md

### [REJECT] wal-replay-filters-commit-checkpoint
Duplicate of already-accepted `wal-replay-ignores-commit`
- Source: entries/2026/05/28/topic-wal-commit-semantics.md

### [REJECT] replay-excludes-commit-and-checkpoint
Duplicate of already-accepted `wal-replay-ignores-commit`
- Source: entries/2026/05/28/topic-wal-crash-recovery-correctness.md

### [REJECT] append-batch-always-force-syncs
Duplicate of already-accepted `wal-batch-always-fsyncs`
- Source: entries/2026/05/28/topic-wal-crash-recovery-correctness.md

### [REJECT] iterate-yields-all-record-types
Duplicate of `wal-iterate-preserves-all-ops` proposed above from the earlier entry
- Source: entries/2026/05/28/topic-wal-crash-recovery-correctness.md

### [REJECT] uncommitted-batch-records-discarded-on-replay
Contradicts accepted belief `wal-replay-ignores-commit` and entry 1's analysis; entry 2 describes what a *correct* replay should do, not what the current implementation does — `replay()` filters by record type without batch-grouping logic
- Source: entries/2026/05/28/topic-wal-crash-recovery-correctness.md

### [REJECT] crc-validates-op-key-value-not-header
Largely duplicate of already-accepted `wal-crc-does-not-cover-seqnum`; the positive framing (what IS covered) is the complement of the existing negative framing
- Source: entries/2026/05/28/topic-wal-crash-recovery-correctness.md

### [ACCEPT] seq-num-corruption-is-silent
A bit flip in a WAL record's `seq_num` passes CRC validation and is silently accepted with the wrong sequence number, potentially corrupting replay filtering, truncation boundaries, and recovery sequence numbering
- Source: entries/2026/05/28/topic-wal-crc-coverage-gap.md

### [ACCEPT] record-length-implicitly-validated
Corruption of `record_length` causes `_read_record` to consume wrong bytes for the payload, which fails the CRC check indirectly — making explicit CRC coverage of the framing field unnecessary
- Source: entries/2026/05/28/topic-wal-crc-coverage-gap.md

### [ACCEPT] all-three-implementations-use-payload-only-crc
The WAL, Bitcask, and B-tree storage engine all compute CRC over data payloads only, excluding their respective header/framing metadata fields — a consistent pattern across the repo
- Source: entries/2026/05/28/topic-wal-crc-coverage-gap.md

### [REJECT] wal-crc-covers-payload-only
Largely duplicate of already-accepted `wal-crc-does-not-cover-seqnum`; the cross-cutting claim is captured by `all-three-implementations-use-payload-only-crc` instead
- Source: entries/2026/05/28/topic-wal-crc-coverage-gap.md

### [ACCEPT] wal-truncate-deletes-oldest-first
`truncate()` iterates segments via `_wal_files()` in oldest-first order, so a crash mid-truncation leaves a contiguous suffix of segments — preserving the recovery invariant that surviving files form a continuous sequence
- Source: entries/2026/05/28/topic-wal-segment-deletion-ordering.md

### [ACCEPT] wal-files-sorted-lexicographic
`_wal_files()` returns segment files sorted by filename (`{N:06d}.wal`), which equals chronological order since segment numbers increment monotonically
- Source: entries/2026/05/28/topic-wal-segment-deletion-ordering.md

### [ACCEPT] wal-no-directory-fsync
The WAL implementation never fsyncs the parent directory after segment creation or deletion, meaning file metadata changes (including unlinks) are not guaranteed durable on crash on Linux
- Source: entries/2026/05/28/topic-wal-segment-deletion-ordering.md

### [ACCEPT] wal-truncate-closes-write-fd
`truncate()` flushes, fsyncs, and closes the current write file descriptor before iterating segments, preventing conflicts with files it may need to delete or rewrite
- Source: entries/2026/05/28/topic-wal-segment-deletion-ordering.md

### [ACCEPT] wal-batch-adds-commit-record
`append_batch` of N items writes N+1 records to the WAL: the N data records plus a trailing record with `op_type == "COMMIT"` that seals the batch
- Source: entries/2026/05/28/write-ahead-log-test_wal.md

### [REJECT] wal-corruption-truncates-at-boundary
Duplicate of already-accepted `wal-crc-mismatch-halts-all-replay`; both describe the same behavior where corrupted records stop replay and only valid prefix records are returned
- Source: entries/2026/05/28/write-ahead-log-test_wal.md

### [ACCEPT] wal-seq-starts-at-one
Sequence numbers begin at 1 for the first appended record; an empty WAL reports `current_seq_num() == 0`
- Source: entries/2026/05/28/write-ahead-log-test_wal.md

### [ACCEPT] wal-truncate-is-inclusive
`truncate(seq)` removes all records with sequence number less than or equal to `seq`, keeping only records where `seq_num > seq`
- Source: entries/2026/05/28/write-ahead-log-test_wal.md

### [ACCEPT] wal-checkpoint-seq-is-next-after-data
Checkpoint markers consume a sequence number in the same monotonic space as data records — a checkpoint after 7 data records occupies seq=8
- Source: entries/2026/05/28/write-ahead-log-test_wal.md


### [ACCEPT] wal-checkpoint-consumes-sequence
`checkpoint()` increments the WAL sequence counter by 1, occupying a position in the sequence number space alongside data records (e.g., after 6 data records at seq 1-6, `current_seq_num()` is 7 and `checkpoint()` returns 8).
- Source: entries/2026/05/28/write-ahead-log-tester_test_wal.md

### [REJECT] wal-replay-excludes-commit-markers
Duplicates existing belief `wal-replay-ignores-commit`.
- Source: entries/2026/05/28/write-ahead-log-tester_test_wal.md

### [ACCEPT] wal-corruption-returns-valid-prefix
When the WAL encounters a corrupted record during replay, it stops and returns all structurally valid records preceding the corruption point rather than raising an exception to the caller.
- Source: entries/2026/05/28/write-ahead-log-tester_test_wal.md

### [REJECT] wal-batch-appends-three-ops-plus-commit
Too test-specific; the general principle (batch appends a trailing COMMIT) is captured by `wal-batch-commit-sentinel` below.
- Source: entries/2026/05/28/write-ahead-log-tester_test_wal.md

### [ACCEPT] wal-sync-mode-survives-crash
With `sync_mode="sync"`, WAL records are durable immediately after `append` returns — they survive process death even if `close()` is never called.
- Source: entries/2026/05/28/write-ahead-log-tester_test_wal.md

### [REJECT] wal-crc-covers-op-key-value-only
Duplicates existing belief `wal-crc-does-not-cover-seqnum`.
- Source: entries/2026/05/28/write-ahead-log-wal-_encode_record.md

### [ACCEPT] wal-record-length-excludes-own-prefix
The `record_length` field counts 21 + len(key) + len(value) bytes — everything after the 4-byte length prefix itself — so total on-disk size per record is `record_length + 4`.
- Source: entries/2026/05/28/write-ahead-log-wal-_encode_record.md

### [ACCEPT] wal-encode-is-pure
`_encode_record` is a pure function that performs no I/O or state mutation; all disk writes (and fsync) are the responsibility of callers (`append`, `append_batch`, `checkpoint`, `truncate`).
- Source: entries/2026/05/28/write-ahead-log-wal-_encode_record.md

### [ACCEPT] wal-key-value-length-signed-int32
Key and value lengths in the WAL binary format are packed as signed 32-bit integers (`struct` format `<i`), limiting each field to ~2 GB and leaving negative lengths unguarded.
- Source: entries/2026/05/28/write-ahead-log-wal-_encode_record.md

### [ACCEPT] wal-little-endian-hardcoded
The WAL binary record format uses little-endian byte order unconditionally (struct prefix `<`), making log files non-portable across architectures with different endianness.
- Source: entries/2026/05/28/write-ahead-log-wal-_encode_record.md

### [REJECT] wal-read-record-none-on-partial
Duplicates existing belief `wal-partial-read-is-eof`.
- Source: entries/2026/05/28/write-ahead-log-wal-_read_record.md

### [REJECT] wal-crc-excludes-seq-num
Duplicates existing belief `wal-crc-does-not-cover-seqnum`.
- Source: entries/2026/05/28/write-ahead-log-wal-_read_record.md

### [ACCEPT] wal-record-format-length-prefixed
Each WAL record on disk is prefixed with a 4-byte little-endian uint32 length covering everything after itself, allowing the reader to atomically skip or validate entire records.
- Source: entries/2026/05/28/write-ahead-log-wal-_read_record.md

### [ACCEPT] wal-read-record-stateless
`_read_record` is a module-level function with no dependency on `WriteAheadLog` instance state, enabling its use during WAL construction/recovery before the object is fully initialized.
- Source: entries/2026/05/28/write-ahead-log-wal-_read_record.md

### [ACCEPT] wal-batch-commit-sentinel
Every `append_batch` call writes exactly one COMMIT record (op code 3) as the final record in the batch, with empty key and value fields, consuming one additional sequence number.
- Source: entries/2026/05/28/write-ahead-log-wal-append_batch.md

### [REJECT] wal-batch-forces-fsync
Duplicates existing belief `wal-batch-always-fsyncs`.
- Source: entries/2026/05/28/write-ahead-log-wal-append_batch.md

### [ACCEPT] wal-batch-single-write
All records in an `append_batch` call (operations + COMMIT) are buffered into a single `bytearray` and written in one `write()` syscall to minimize the window for partial writes.
- Source: entries/2026/05/28/write-ahead-log-wal-append_batch.md

### [REJECT] wal-replay-ignores-commit-boundaries
Duplicates existing belief `wal-replay-ignores-commit`.
- Source: entries/2026/05/28/write-ahead-log-wal-append_batch.md

### [ACCEPT] wal-no-rollback-on-io-failure
If `write()` or `fsync()` fails mid-batch in `append_batch`, sequence numbers are already incremented with no rollback mechanism, creating a permanent gap in the sequence space.
- Source: entries/2026/05/28/write-ahead-log-wal-append_batch.md

### [ACCEPT] iterate-yields-all-op-types
`iterate()` yields records of every op type (PUT, DELETE, COMMIT, CHECKPOINT) while `replay()` filters to only PUT and DELETE records.
- Source: entries/2026/05/28/write-ahead-log-wal-iterate.md

### [ACCEPT] iterate-not-snapshot-isolated
`iterate()` releases the WAL lock after flushing but before reading begins, so concurrent appends may or may not be visible to the iterator depending on timing.
- Source: entries/2026/05/28/write-ahead-log-wal-iterate.md

### [ACCEPT] iterate-stops-silently-at-corruption
When `iterate()` encounters a CRC mismatch, iteration stops with no exception or sentinel value — the caller receives all valid records before the corruption with no indication that more data existed.
- Source: entries/2026/05/28/write-ahead-log-wal-iterate.md

### [ACCEPT] iterate-flushes-without-fsync
Before reading, `iterate()` calls `fd.flush()` but not `os.fsync()`, so buffered writes reach the OS page cache but are not guaranteed durable on stable storage.
- Source: entries/2026/05/28/write-ahead-log-wal-iterate.md


### [REJECT] replay-returns-only-put-delete
Duplicates existing `wal-replay-ignores-commit` — both describe filtering out non-data records during replay
- Source: entries/2026/05/28/write-ahead-log-wal-replay.md

### [REJECT] replay-stops-at-corruption
Duplicates existing `wal-crc-mismatch-halts-all-replay` — same claim that CRC failure terminates replay for all subsequent records
- Source: entries/2026/05/28/write-ahead-log-wal-replay.md

### [ACCEPT] replay-does-not-enforce-batch-atomicity
Despite the docstring claiming replay "skips uncommitted batches," the implementation returns all PUT/DELETE records regardless of whether a matching COMMIT record exists — a crash mid-batch will replay partial batch records
- Source: entries/2026/05/28/write-ahead-log-wal-replay.md

### [ACCEPT] replay-flush-before-read
`replay` flushes the write file descriptor under `self._lock` before reading, ensuring all prior `append` calls are visible on disk during the read phase
- Source: entries/2026/05/28/write-ahead-log-wal-replay.md

### [ACCEPT] replay-not-lock-protected-during-read
The file read phase of `replay` runs without holding `self._lock`, so concurrent `append` calls during replay could produce a partial read of in-flight writes
- Source: entries/2026/05/28/write-ahead-log-wal-replay.md

### [ACCEPT] replay-strict-greater-than-filter
`replay(after_seq=n)` returns records with `seq_num` strictly greater than `n` (exclusive lower bound), not greater-or-equal — passing 0 replays the entire log
- Source: entries/2026/05/28/write-ahead-log-wal-replay.md

### [ACCEPT] replay-returns-eager-list
`replay` returns an in-memory `List[WALRecord]` snapshot, not a lazy iterator — the full result set is materialized before returning, which can be large if the WAL is large
- Source: entries/2026/05/28/write-ahead-log-wal-replay.md

### [REJECT] wal-truncate-not-atomic
Duplicates existing `wal-truncate-not-crash-safe` — both describe the lack of atomic file operations (no temp-file-rename pattern) during truncation
- Source: entries/2026/05/28/write-ahead-log-wal-truncate.md

### [ACCEPT] wal-truncate-inclusive-boundary
`truncate(n)` removes records with `seq_num <= n` (inclusive upper bound), keeping only records strictly greater than `n`
- Source: entries/2026/05/28/write-ahead-log-wal-truncate.md

### [ACCEPT] wal-truncate-drops-after-corruption
During truncate, if a corrupt record is encountered in a WAL file, all subsequent records in that file are silently discarded regardless of their sequence number — even records with `seq_num > up_to_seq`
- Source: entries/2026/05/28/write-ahead-log-wal-truncate.md

### [ACCEPT] wal-truncate-deletes-empty-files
WAL files where every record has `seq_num <= up_to_seq` are deleted entirely via `os.remove` rather than left as empty files on disk
- Source: entries/2026/05/28/write-ahead-log-wal-truncate.md

### [ACCEPT] wal-truncate-blocks-all-operations
`truncate` holds `self._lock` for its entire duration (close, rewrite, reopen), blocking all concurrent appends, replays, and iterates
- Source: entries/2026/05/28/write-ahead-log-wal-truncate.md

### [ACCEPT] wal-truncate-no-bounds-check
`truncate` does not validate that `up_to_seq` is within range — passing a value beyond `current_seq_num()` silently deletes all records without error
- Source: entries/2026/05/28/write-ahead-log-wal-truncate.md



---

**Generated:** 2026-05-29
**Source:** 37 entries from entries/
**Model:** claude

### [ACCEPT] es-event-ids-are-stream-scoped
Event IDs are sequential integers per-stream starting at 1, not globally unique identifiers; `global_position` is the separate cross-stream counter.
- Source: entries/2026/05/29/event-sourcing-store-test_event_store.md

### [ACCEPT] es-global-position-tracks-all-streams
`global_position` increments monotonically across all streams and equals the total number of events ever appended to the store.
- Source: entries/2026/05/29/event-sourcing-store-test_event_store.md

### [ACCEPT] es-live-projection-auto-updates
`LiveProjection` automatically applies new events on each `append()` after the initial `catch_up()` call — no subsequent explicit `catch_up()` required.
- Source: entries/2026/05/29/event-sourcing-store-test_event_store.md

### [ACCEPT] es-projection-catch-up-is-incremental
`Projection.catch_up()` only processes events after its current `position`, returning the count of newly processed events; a second call with no new events processes zero.
- Source: entries/2026/05/29/event-sourcing-store-test_event_store.md

### [ACCEPT] es-persistence-uses-jsonl
EventStore disk persistence uses JSONL (newline-delimited JSON) format, loaded in full on construction to rebuild the in-memory state.
- Source: entries/2026/05/29/event-sourcing-store-test_event_store.md

### [ACCEPT] es-optimistic-concurrency-via-expected-version
`append(..., expected_version=V)` succeeds only if the current stream version equals V; a stale version raises `ConcurrencyConflict`.
- Source: entries/2026/05/29/event-sourcing-store-test_event_store.md

### [ACCEPT] es-snapshot-saves-state-and-position
`Projection.save_snapshot()` persists both the projection state and its current position; `load_snapshot()` restores both so that `catch_up()` resumes from the snapshot point.
- Source: entries/2026/05/29/event-sourcing-store-test_event_store.md

### [ACCEPT] es-temporal-query-via-up-to
`reconstruct_state(handlers, events, up_to=N)` replays exactly the first N events to rebuild past state at that point in time.
- Source: entries/2026/05/29/event-sourcing-store-test_event_store.md

### [REJECT] hint-files-only-from-compaction
Duplicate of existing belief `hint-written-only-during-compaction`.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_load_hint_file.md

### [REJECT] hint-no-tombstones
Duplicate of existing belief `bitcask-hint-files-exclude-tombstones`.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_load_hint_file.md

### [ACCEPT] hint-format-variable-length
Each hint record is `HINT_HEADER_SIZE (24 bytes) + 4 (key_size uint32) + key_length` bytes; the `HINT_FORMAT` constant covers only the fixed 24-byte portion, not the full record.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_load_hint_file.md

### [REJECT] hint-no-integrity-check
Duplicate of existing belief `hint-file-no-integrity-check`.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_load_hint_file.md

### [REJECT] rebuild-order-matters
Duplicate of existing beliefs `bitcask-recovery-order-is-ascending` and `bitcask-rebuild-sorts-ascending`.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_load_hint_file.md

### [REJECT] bitcask-tombstone-is-empty-value
Duplicate of existing belief `bitcask-tombstone-is-empty-string`.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_scan_data_file.md

### [REJECT] bitcask-scan-order-determines-correctness
Duplicate of existing beliefs `bitcask-recovery-order-is-ascending` and `scan-segment-ordering-invariant`.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_scan_data_file.md

### [ACCEPT] bitcask-scan-short-payload-undetected
In `_scan_data_file`, a truncated header safely breaks the scan loop, but a short key/value read after a valid header is not detected — the truncated data is silently used, potentially corrupting the keydir.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_scan_data_file.md

### [REJECT] tester-bitcask-sync-writes-disabled
Duplicate of existing belief `bitcask-tests-disable-sync`.
- Source: entries/2026/05/29/hash-index-storage-tester_test_bitcask.md

### [REJECT] tester-bitcask-recovery-two-paths
Covered by existing beliefs `bitcask-crash-recovery-without-hints` and `bitcask-hint-files-skip-scan`.
- Source: entries/2026/05/29/hash-index-storage-tester_test_bitcask.md

### [REJECT] tester-bitcask-compaction-removes-tombstones
Duplicate of existing belief `compact-purges-tombstones`.
- Source: entries/2026/05/29/hash-index-storage-tester_test_bitcask.md

### [ACCEPT] bitcask-file-rotation-transparent
File rotation triggered by `max_file_size` is invisible to readers — all keys remain retrievable across multiple `.data` files because the keydir maps each key to its specific `(file_id, offset)`.
- Source: entries/2026/05/29/hash-index-storage-tester_test_bitcask.md

### [REJECT] tester-bitcask-delete-nonexistent-safe
Duplicate of existing belief `bitcask-delete-nonexistent-is-noop`.
- Source: entries/2026/05/29/hash-index-storage-tester_test_bitcask.md

### [REJECT] recover-replay-order
Duplicate of existing belief `bitcask-recovery-order-is-ascending`.
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_recover.md

### [REJECT] hint-files-skip-crc
Duplicate of existing belief `hint-file-no-integrity-check`.
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_recover.md

### [REJECT] recover-discards-trailing-corruption
Covered by existing beliefs `corruption-terminates-scan` and `bitcask-partial-write-safe`.
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_recover.md

### [REJECT] no-concurrent-writer-protection
Duplicate of existing belief `bitcask-single-active-writer`.
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_recover.md

### [REJECT] active-segment-is-highest-id
Covered by existing belief `bitcask-compaction-renumbers-active`.
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_recover.md


### [REJECT] hint-files-skip-tombstones
Duplicates existing `bitcask-hint-files-exclude-tombstones`
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-create_hint_files.md

### [ACCEPT] hint-files-only-canonical-entries
A hint entry is written only when `self._index[key]` confirms the key's canonical location is the current segment and offset, filtering out entries superseded by later writes or compaction
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-create_hint_files.md

### [ACCEPT] hint-generation-skips-active-segment
`create_hint_files` never generates a hint file for the active segment, only for frozen (immutable) segments, since a hint for the active segment would be immediately stale
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-create_hint_files.md

### [ACCEPT] hint-generation-no-crc-validation
`create_hint_files` does not verify CRC checksums on the segment records it reads (unlike `_scan_segment`), so corrupted data can be silently indexed via hint files
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-create_hint_files.md

### [ACCEPT] hint-file-write-not-atomic
Hint files are written directly to the final path (not via temp-file-and-rename), so a crash mid-generation leaves a partial hint file; `_load_hint_file` handles this by stopping at the first short read
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-create_hint_files.md

### [ACCEPT] put-append-only
`put` never modifies existing on-disk data; overwrites create a new record appended to the active segment and update only the in-memory index, leaving old records as reclaimable garbage until compaction
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-put.md

### [ACCEPT] put-no-tombstone-guard
`put` does not reject the `TOMBSTONE` sentinel as a value; passing it creates a record indistinguishable from a delete, which silently corrupts the key's state on recovery
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-put.md

### [REJECT] put-triggers-compaction
Substantially overlaps with existing `bitcask-auto-compact-threshold` which already captures the auto-compaction trigger behavior
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-put.md

### [REJECT] put-not-thread-safe
Covered by existing `no-concurrency-primitives` which applies to the entire codebase
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-put.md

### [REJECT] put-crash-recovery-gap
The crash window between `_write_record` and index update is already covered by `bitcask-partial-write-safe` and `bitcask-crash-recovery-without-hints`
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-put.md

### [ACCEPT] tester-tests-public-contract
`tester_test_*.py` files test the public API contract of each implementation, importing only public symbols (plus format constants for fault injection), acting as external conformance harnesses
- Source: entries/2026/05/29/log-structured-hash-table-tester_test_bitcask.md

### [REJECT] bitcask-get-returns-none-for-missing
Duplicates existing `bitcask-get-missing-returns-none`
- Source: entries/2026/05/29/log-structured-hash-table-tester_test_bitcask.md

### [ACCEPT] bitcask-crc32-raises-corruption-error
Reading a record whose payload does not match its CRC32 header raises `CorruptionError` — the store never silently returns corrupt data
- Source: entries/2026/05/29/log-structured-hash-table-tester_test_bitcask.md

### [REJECT] bitcask-partial-write-silent-discard
Overlaps with existing `bitcask-partial-write-safe` which covers the same recovery behavior
- Source: entries/2026/05/29/log-structured-hash-table-tester_test_bitcask.md

### [REJECT] bitcask-compaction-preserves-live-data
Duplicates existing `bitcask-compaction-preserves-observable-state`
- Source: entries/2026/05/29/log-structured-hash-table-tester_test_bitcask.md

### [ACCEPT] flush-creates-sequential-sstables
Each `_flush` call creates an SSTable file with a strictly increasing sequence number (zero-padded filename for lexicographic = numeric sort), preserving newest-last ordering in `self._sstables`
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-_flush.md

### [REJECT] wal-truncated-after-sstable-write
Duplicates existing `lsm-wal-truncated-after-sstable-write`
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-_flush.md

### [ACCEPT] flush-skips-immutable-memtable-stage
`_flush` writes the frozen memtable directly to an SSTable without staging it in `_immutable_memtables`, creating a brief window where in-flight keys are invisible to `get()` despite `_immutable_memtables` being checked in the read path
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-_flush.md

### [ACCEPT] compaction-triggered-by-sstable-count
LSM compaction runs automatically when `len(self._sstables) >= self._compaction_threshold` (default 4), triggered at the end of `_flush` after the new SSTable is registered
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-_flush.md

### [ACCEPT] multi-leader-lww-deterministic-tiebreak
LWW conflict resolution compares `(timestamp, node_id)` tuples using Python's lexicographic tuple comparison, guaranteeing all nodes independently reach the same winner without coordination
- Source: entries/2026/05/29/multi-leader-replication-multi_leader.md

### [ACCEPT] multi-leader-tombstone-delete
Deletes are implemented as tombstone writes (`_TOMBSTONE` sentinel with `is_tombstone=True`) that replicate like normal mutations; `get()` returns `None` for tombstoned keys
- Source: entries/2026/05/29/multi-leader-replication-multi_leader.md

### [ACCEPT] multi-leader-custom-merge-new-timestamp
Custom merge resolution creates a new timestamp (`max(local_ts, remote_ts) + 1`) and canonical origin so the merged result supersedes both conflicting inputs in any future LWW comparison
- Source: entries/2026/05/29/multi-leader-replication-multi_leader.md

### [ACCEPT] multi-leader-idempotent-apply
Each `(key, timestamp, origin_node)` triple is applied at most once per node, tracked by the `_seen` set, preventing duplicate application in ring topologies where changes propagate through multiple hops
- Source: entries/2026/05/29/multi-leader-replication-multi_leader.md

### [ACCEPT] multi-leader-sync-collects-then-distributes
`sync()` drains all pending queues from every node before distributing any changes, preventing intra-round cascading where one node's change would trigger further propagation within the same sync round
- Source: entries/2026/05/29/multi-leader-replication-multi_leader.md


### [ACCEPT] sstable-get-miss-returns-none
`SSTableReader.get()` returns `None` for keys not found in the table, rather than raising an exception
- Source: entries/2026/05/29/sstable-and-compaction-tester_test_sstable.md

### [ACCEPT] sstable-merge-accepts-reader-objects
`merge_sstables()` takes `SSTableReader` instances directly as inputs, using a streaming iterator interface rather than loading entire files into memory
- Source: entries/2026/05/29/sstable-and-compaction-tester_test_sstable.md

### [ACCEPT] size-tiered-triggers-on-sstable-count
Size-tiered compaction triggers when the total SSTable count meets the `min_threshold` parameter
- Source: entries/2026/05/29/sstable-and-compaction-tester_test_sstable.md

### [ACCEPT] leveled-triggers-on-l0-count
Leveled compaction triggers when the L0 SSTable count meets the `l0_compaction_trigger` parameter
- Source: entries/2026/05/29/sstable-and-compaction-tester_test_sstable.md

### [REJECT] sstable-range-scan-exclusive-end
Duplicates existing belief `sstable-range-scan-end-exclusive`
- Source: entries/2026/05/29/sstable-and-compaction-tester_test_sstable.md

### [REJECT] sstable-timestamp-conflict-resolution
Duplicates existing belief `sstable-merge-dedup-newest-timestamp`
- Source: entries/2026/05/29/sstable-and-compaction-tester_test_sstable.md

### [REJECT] sstable-tombstone-optional-removal
Duplicates existing belief `merge-preserves-tombstones-by-default`
- Source: entries/2026/05/29/sstable-and-compaction-tester_test_sstable.md

### [REJECT] sstable-leveled-compaction-promotes-to-l1
Duplicates existing belief `leveled-compaction-promotes-to-l1`
- Source: entries/2026/05/29/sstable-and-compaction-tester_test_sstable.md

### [REJECT] sstable-tester-tests-happy-path-only
Meta-observation about test coverage, not a testable architectural invariant about the codebase itself
- Source: entries/2026/05/29/sstable-and-compaction-tester_test_sstable.md

### [REJECT] compaction-no-manifest
Duplicates existing belief `no-compaction-manifest`
- Source: entries/2026/05/29/topic-bitcask-crash-safety.md

### [REJECT] recovery-tolerates-partial-writes
Duplicates existing belief `bitcask-partial-write-safe` and `corruption-terminates-scan`
- Source: entries/2026/05/29/topic-bitcask-crash-safety.md

### [REJECT] sstable-header-written-twice
Duplicates existing belief `sstable-writer-append-then-patch`
- Source: entries/2026/05/29/topic-bitcask-crash-safety.md

### [REJECT] delete-before-rename-in-bitcask
Duplicates existing belief `delete-before-rename-ordering`
- Source: entries/2026/05/29/topic-bitcask-crash-safety.md

### [REJECT] no-fsync-on-compaction-output
Covered by existing beliefs `lsm-compaction-deletes-without-fsync-barrier` and `hint-file-no-fsync`
- Source: entries/2026/05/29/topic-bitcask-crash-safety.md

### [ACCEPT] partial-hint-load-truncates-silently
`_load_hint_file` in both Bitcask implementations treats a short read as end-of-file (breaks out of the read loop without error), silently dropping all keys that would have appeared after the truncation point
- Source: entries/2026/05/29/topic-bitcask-hint-durability.md

### [REJECT] hint-file-no-checksum
Duplicates existing belief `hint-file-no-integrity-check`
- Source: entries/2026/05/29/topic-bitcask-hint-durability.md

### [REJECT] hint-presence-suppresses-scan
Duplicates existing belief `bitcask-hint-files-skip-scan`
- Source: entries/2026/05/29/topic-bitcask-hint-durability.md

### [REJECT] orphaned-data-unreachable
Duplicates existing belief `hint-file-orphan-crash-risk`
- Source: entries/2026/05/29/topic-bitcask-hint-durability.md

### [ACCEPT] log-structured-index-omits-size-and-timestamp
The `log-structured-hash-table` Bitcask variant stores only `(filepath, offset)` in its index, omitting record size and timestamp, requiring an extra header parse on every `get()` to discover record boundaries
- Source: entries/2026/05/29/topic-bitcask-paper-design.md

### [ACCEPT] log-structured-hint-creation-is-manual
In `log-structured-hash-table/bitcask.py`, hint file creation requires an explicit `create_hint_files()` call rather than being produced automatically as a side-effect of compaction
- Source: entries/2026/05/29/topic-bitcask-paper-design.md

### [ACCEPT] log-structured-bitcask-no-fsync
`log-structured-hash-table/bitcask.py` uses only `flush()` without `os.fsync()` on data writes, unlike `hash-index-storage` which calls `os.fsync` per record when `sync_writes` is enabled
- Source: entries/2026/05/29/topic-bitcask-paper-design.md

### [REJECT] hint-files-produced-by-merge-only
Duplicates existing belief `hint-written-only-during-compaction`
- Source: entries/2026/05/29/topic-bitcask-paper.md


### [ACCEPT] bitcask-no-mmap
Neither Bitcask implementation uses memory-mapped I/O; both use standard `open()`/`read()` with manual buffering, forgoing kernel page cache optimizations that production Bitcask implementations rely on for startup performance
- Source: entries/2026/05/29/topic-bitcask-startup-cost.md

### [ACCEPT] bitcask-no-parallel-rebuild
Both Bitcask implementations process data/hint files strictly sequentially during keydir rebuild with no threading, multiprocessing, or async I/O, even though hint file loading is embarrassingly parallel
- Source: entries/2026/05/29/topic-bitcask-startup-cost.md

### [ACCEPT] hint-format-diverges-between-implementations
Hash-index hint uses a 24-byte fixed header per entry (`file_id:u32, offset:u64, size:u32, timestamp:f64`) while log-structured hint uses only an 8-byte header (`key_size:u32, offset:u32`), omitting file_id, timestamp, and record size
- Source: entries/2026/05/29/topic-bitcask-startup-cost.md

### [ACCEPT] log-structured-hint-creation-decoupled
The log-structured Bitcask exposes `create_hint_files()` as a standalone operation callable independently of compaction, while the hash-index variant only writes hint files inside `compact()`
- Source: entries/2026/05/29/topic-bitcask-startup-cost.md

### [ACCEPT] data-scan-skips-value-bytes
`_scan_data_file` in the hash-index Bitcask reads each record's header and key but seeks past value bytes; it is still O(data_size) because headers must be parsed sequentially to locate record boundaries
- Source: entries/2026/05/29/topic-bitcask-startup-cost.md

### [ACCEPT] lsm-wal-truncate-destroys-immediately
The LSM tree's `WAL.truncate()` opens the file with `"wb"` mode which zeroes all content instantly, with no intermediate durable state to recover from if a crash occurs before the file is reopened for appending
- Source: entries/2026/05/29/topic-crash-safety-of-truncate.md

### [ACCEPT] checkpoint-record-unused-for-truncation
The WAL supports `OP_CHECKPOINT` records that could serve as a durable truncation watermark (skip records below the watermark during replay), but `truncate()` physically removes records instead of using this mechanism
- Source: entries/2026/05/29/topic-crash-safety-of-truncate.md

### [ACCEPT] crc-detects-not-prevents-data-loss
CRC32 checksums in the WAL and Bitcask detect partial writes after a crash but cannot recover the lost data — they convert silent corruption into detected data loss, not into a recoverable state
- Source: entries/2026/05/29/topic-crash-safety-of-truncate.md

### [ACCEPT] btree-wal-uses-trailer-crc
The B-tree WAL places its CRC32 checksum as a trailer after `page_data`, while the standalone WAL and Bitcask embed CRC in the record header — a structural difference that affects torn-write detection behavior
- Source: entries/2026/05/29/topic-crc-scope-comparison-across-implementations.md

### [ACCEPT] wal-crc-includes-op-type
The standalone WAL uniquely includes `op_type_byte` in its CRC input alongside key and value, protecting against silent operation-type corruption (e.g., PUT flipped to DELETE) that the other CRC implementations leave undetected
- Source: entries/2026/05/29/topic-crc-scope-comparison-across-implementations.md

### [REJECT] hash-index-bitcask-has-no-integrity-check
Duplicate of existing belief `bitcask-no-checksum-validation`
- Source: entries/2026/05/29/topic-crc-scope-comparison-across-implementations.md

### [ACCEPT] merkle-tree-uses-sha256-for-cross-replica-verification
The Merkle tree uses SHA-256 (`merkle_tree.py:11`) for tamper-evident cross-replica verification via `verify_proof`, while all storage engines use CRC32 — reflecting the split between accidental-corruption detection on trusted local disk and adversarial-integrity across trust boundaries
- Source: entries/2026/05/29/topic-crc32-vs-cryptographic-hash.md

### [REJECT] sstable-has-no-integrity-check
Duplicate of existing belief `sstable-no-checksums`
- Source: entries/2026/05/29/topic-crc32-vs-cryptographic-hash.md

### [REJECT] hash-index-bitcask-lacks-crc
Duplicate of existing belief `bitcask-no-checksum-validation`
- Source: entries/2026/05/29/topic-crc32-vs-cryptographic-hash.md

### [REJECT] wal-crc-covers-payload-not-header
Duplicate of existing beliefs `wal-protects-data-not-metadata` and `wal-crc-does-not-cover-seqnum`
- Source: entries/2026/05/29/topic-crc32-vs-cryptographic-hash.md

### [REJECT] no-dir-fsync-after-unlink
Duplicate of existing belief `no-directory-fsync-anywhere`
- Source: entries/2026/05/29/topic-directory-fsync-after-unlink.md

### [REJECT] all-fsyncs-target-file-fds
Logically equivalent to existing belief `no-directory-fsync-anywhere`
- Source: entries/2026/05/29/topic-directory-fsync-after-unlink.md

### [REJECT] compaction-deletion-not-crash-safe
Duplicate of existing beliefs `lsm-compaction-deletes-without-fsync-barrier` and `bitcask-compact-not-crash-safe`
- Source: entries/2026/05/29/topic-directory-fsync-after-unlink.md

### [REJECT] rename-not-crash-durable
Duplicate of existing belief `rename-without-dir-barrier`
- Source: entries/2026/05/29/topic-directory-fsync-after-unlink.md


### [ACCEPT] btree-page-overwrite-no-size-change
B-tree PageManager overwrites pages at fixed offsets within a pre-allocated file, so most write+sync cycles do not change file size and would benefit from `fdatasync` skipping metadata I/O
- Source: entries/2026/05/29/topic-fdatasync-vs-fsync-tradeoffs.md

### [ACCEPT] all-fsync-sites-data-integrity-only
None of the 13 `os.fsync()` call sites have callers that depend on mtime or ctime metadata being accurate; all syncs exist purely for data durability, making every site a valid `fdatasync` candidate
- Source: entries/2026/05/29/topic-fdatasync-vs-fsync-tradeoffs.md

### [REJECT] lsm-wal-missing-fsync
Duplicate of existing belief `lsm-wal-has-no-fsync`
- Source: entries/2026/05/29/topic-fdatasync-vs-fsync-tradeoffs.md

### [REJECT] no-directory-fsync-after-file-creation
Duplicate of existing belief `no-directory-fsync-anywhere`
- Source: entries/2026/05/29/topic-fdatasync-vs-fsync-tradeoffs.md

### [ACCEPT] batch-mode-defers-fsync
In `batch` sync mode, individual WAL appends do not fsync until `_write_count` reaches `_batch_sync_count` (default 100), leaving up to 99 records vulnerable to crash loss
- Source: entries/2026/05/29/topic-fsync-semantics-by-mode.md

### [ACCEPT] none-mode-never-syncs-on-append
In `none` sync mode, `_do_sync()` is a complete no-op for individual appends; data reaches stable storage only via WAL rotation or explicit close
- Source: entries/2026/05/29/topic-fsync-semantics-by-mode.md

### [ACCEPT] force-true-bypasses-sync-mode
Passing `force=True` to `_do_sync()` causes flush+fsync regardless of the configured sync mode, enabling group commit at batch boundaries
- Source: entries/2026/05/29/topic-fsync-semantics-by-mode.md

### [ACCEPT] rotation-always-fsyncs
The WAL `_rotate()` method unconditionally calls `flush()` and `os.fsync()` before closing the current segment, providing a durability checkpoint even in `none` mode
- Source: entries/2026/05/29/topic-fsync-semantics-by-mode.md

### [ACCEPT] batch-write-count-not-reset-on-forced-sync
The WAL `_write_count` counter only resets when the batch threshold triggers a sync (line 133), not when a forced sync occurs, which may cause counter drift between forced and threshold-triggered syncs
- Source: entries/2026/05/29/topic-fsync-semantics-by-mode.md

### [REJECT] lsm-compact-removes-all-tombstones
Covered by existing beliefs `lsm-compact-removes-tombstones-safely` and `lsm-compaction-is-full-merge` together
- Source: entries/2026/05/29/topic-leveled-compaction-tombstone-policy.md

### [ACCEPT] merge-sstables-tombstone-flag-is-caller-controlled
`merge_sstables()` accepts a `remove_tombstones` boolean from the caller rather than computing safety from level metadata, so correctness depends entirely on the caller passing the right value
- Source: entries/2026/05/29/topic-leveled-compaction-tombstone-policy.md

### [ACCEPT] sstable-compaction-manager-lacks-bottommost-detection
The `CompactionManager` tracks `_max_levels` (default 7) but has no logic to check whether lower levels contain overlapping key ranges before deciding tombstone removal, unlike LevelDB/RocksDB which use `VersionSet` metadata for this
- Source: entries/2026/05/29/topic-leveled-compaction-tombstone-policy.md

### [REJECT] premature-tombstone-removal-causes-data-resurrection
General correctness principle already covered by existing belief `tombstone-removal-requires-full-key-coverage`
- Source: entries/2026/05/29/topic-leveled-compaction-tombstone-policy.md

### [ACCEPT] stcs-overlapping-key-ranges
Size-tiered compaction allows multiple SSTables at the same tier to contain overlapping key ranges, requiring point lookups to check all SSTables in a tier
- Source: entries/2026/05/29/topic-leveled-vs-size-tiered-compaction.md

### [REJECT] leveled-promotes-to-numbered-levels
Duplicate of existing belief `leveled-compaction-promotes-to-l1`
- Source: entries/2026/05/29/topic-leveled-vs-size-tiered-compaction.md

### [REJECT] timestamp-resolves-merge-conflicts
Duplicate of existing belief `sstable-merge-dedup-newest-timestamp`
- Source: entries/2026/05/29/topic-leveled-vs-size-tiered-compaction.md

### [REJECT] lsm-compact-is-full-merge
Duplicate of existing belief `lsm-compaction-is-full-merge`
- Source: entries/2026/05/29/topic-leveled-vs-size-tiered-compaction.md

### [REJECT] sparse-index-bounds-scan-cost
Covered by existing beliefs `sstable-sparse-index-every-nth` and `lsm-sparse-index-default-16`
- Source: entries/2026/05/29/topic-leveled-vs-size-tiered-compaction.md

### [ACCEPT] wal-checkpoint-returns-seq
`WriteAheadLog.checkpoint()` returns the sequence number it wrote, providing the caller the exact truncation boundary for the WAL-to-data-store coordination protocol
- Source: entries/2026/05/29/topic-log-structured-checkpoint-coordination.md

### [ACCEPT] wal-truncate-requires-explicit-seq
`truncate(up_to_seq)` takes an explicit sequence number parameter; the WAL never decides what to truncate on its own — the caller must provide the boundary
- Source: entries/2026/05/29/topic-log-structured-checkpoint-coordination.md

### [ACCEPT] wal-seq-num-recovered-on-init
On construction, `_recover_seq_num()` scans all WAL files to find the maximum surviving sequence number, guaranteeing the monotonic counter never goes backward across crashes
- Source: entries/2026/05/29/topic-log-structured-checkpoint-coordination.md

### [ACCEPT] wal-truncate-fsyncs-before-scan
`truncate()` flushes and fsyncs the current WAL file before scanning for records to remove, preventing data loss from buffered writes that haven't reached disk
- Source: entries/2026/05/29/topic-log-structured-checkpoint-coordination.md

### [ACCEPT] lsm-wal-truncate-no-arg
The LSM tree at `lsm.py:314` calls `self._wal.truncate()` with no sequence argument, indicating it uses a different WAL interface than the standalone `write-ahead-log/wal.py` module
- Source: entries/2026/05/29/topic-log-structured-checkpoint-coordination.md


### [ACCEPT] wal-truncation-vs-corruption-distinction
`_read_record` returns `None` for short reads (truncation) but raises `ValueError` for CRC mismatch (corruption), giving callers two distinct failure modes to handle differently during recovery
- Source: entries/2026/05/29/topic-partial-write-detection.md

### [ACCEPT] wal-no-torn-write-tests
The WAL module has no tests for truncated records, CRC mismatches, or partial writes; the B-tree module tests CRC corruption explicitly but the WAL does not
- Source: entries/2026/05/29/topic-partial-write-detection.md

### [REJECT] wal-record-length-prefix-guards-short-writes
Duplicate of existing `wal-partial-read-is-eof` and `wal-record-length-prefixed`
- Source: entries/2026/05/29/topic-partial-write-detection.md

### [REJECT] lsm-full-compaction-only
Duplicate of existing `lsm-compaction-is-full-merge`
- Source: entries/2026/05/29/topic-sstable-tombstone-gc-safety.md

### [REJECT] sstable-merge-tombstone-opt-in
Duplicate of existing `merge-preserves-tombstones-by-default`
- Source: entries/2026/05/29/topic-sstable-tombstone-gc-safety.md

### [REJECT] bitcask-immutable-only-compaction
Duplicate of existing `bitcask-compact-skips-active-file-keys`
- Source: entries/2026/05/29/topic-sstable-tombstone-gc-safety.md

### [ACCEPT] dynamo-no-delete-support
The leaderless replication implementation (`dynamo.py`) has no delete or tombstone mechanism; adding deletes without tombstones would cause resurrection via read repair or anti-entropy
- Source: entries/2026/05/29/topic-sstable-tombstone-gc-safety.md

### [ACCEPT] multi-leader-no-tombstone-gc
Multi-leader replication stores tombstones indefinitely with no compaction or garbage collection method, which is safe but accumulates dead entries without bound
- Source: entries/2026/05/29/topic-sstable-tombstone-gc-safety.md

### [ACCEPT] tester-files-are-standalone-runnable
Every `tester_test_*.py` file contains an `if __name__ == "__main__":` block, making it executable via `python` without pytest
- Source: entries/2026/05/29/topic-tester-vs-test-convention.md

### [ACCEPT] tester-naming-avoids-pytest-discovery
The `tester_test_*.py` prefix does not match pytest's default collection patterns (`test_*.py` / `*_test.py`), so these files are excluded from `pytest` runs unless explicitly specified
- Source: entries/2026/05/29/topic-tester-vs-test-convention.md

### [ACCEPT] tester-files-use-stdout-validation
Tester files print `"test_name PASSED"` to stdout, indicating an output-parsing runner rather than pytest's exit-code-based reporting
- Source: entries/2026/05/29/topic-tester-vs-test-convention.md

### [ACCEPT] pytest-files-have-broader-coverage
The `test_*.py` files are consistently longer than their `tester_test_*.py` counterparts (e.g., 207 vs 125 lines for bloom-filter), containing additional edge case tests beyond the tester's core invariant checks
- Source: entries/2026/05/29/topic-tester-vs-test-convention.md

### [ACCEPT] tester-pytest-boundary-is-leaky
At least 6 `tester_test_*.py` files import pytest despite being designed for standalone execution, indicating the two suites share lineage rather than being fully independent
- Source: entries/2026/05/29/topic-tester-vs-test-convention.md

### [ACCEPT] wal-concurrent-write-during-read-can-trigger-corruption-stop
A concurrent `append()` writing to a file the iterator is reading can produce a partial record that `_read_record` interprets as CRC failure, terminating the entire iterator and dropping all remaining valid records across subsequent files
- Source: entries/2026/05/29/topic-wal-concurrency-safety.md

### [REJECT] wal-iterate-replay-lock-scope
Duplicate of existing `replay-not-lock-protected-during-read` combined with `replay-flush-before-read`
- Source: entries/2026/05/29/topic-wal-concurrency-safety.md

### [REJECT] lsm-range-scan-zero-concurrency-protection
Duplicate of existing `range-scan-no-concurrency-protection`
- Source: entries/2026/05/29/topic-wal-concurrency-safety.md

### [REJECT] wal-flush-not-fsync-before-iteration
Duplicate of existing `iterate-flushes-without-fsync`
- Source: entries/2026/05/29/topic-wal-concurrency-safety.md

### [ACCEPT] sstable-has-magic-and-version
The SSTable format (`sstable.py:10-11`) uses a 4-byte magic `b"SSTB"` and a 2-byte version field, making it the only binary format in the repo with format versioning
- Source: entries/2026/05/29/topic-wal-record-format-evolution.md

### [ACCEPT] wal-no-resync-after-corruption
A corrupted `record_length` in the WAL causes `_read_record` to misframe all subsequent records; there is no magic-byte or scan-forward recovery mechanism to re-synchronize
- Source: entries/2026/05/29/topic-wal-record-format-evolution.md

### [REJECT] wal-crc-excludes-header
Duplicate of existing `wal-protects-data-not-metadata` and `wal-crc-does-not-cover-seqnum`
- Source: entries/2026/05/29/topic-wal-record-format-evolution.md

### [REJECT] bitcask-no-integrity-check
Contradicts existing accepted belief `bitcask-crc-covers-payload-only` which asserts Bitcask does have a CRC
- Source: entries/2026/05/29/topic-wal-record-format-evolution.md


### [ACCEPT] wal-corruption-stops-per-file
`_recover_seq_num` stops at the first corrupted record within each file but continues to subsequent WAL files, so cross-file recovery is resilient but intra-file records after corruption are silently lost
- Source: entries/2026/05/29/topic-wal-recovery-semantics.md

### [ACCEPT] wal-no-gap-detection
Neither the WAL reader nor any visible consumer checks for sequence number gaps after replay, making silent data loss from mid-file corruption undetectable
- Source: entries/2026/05/29/topic-wal-recovery-semantics.md

### [ACCEPT] wal-batch-single-file
`append_batch` writes the entire batch buffer in one `_fd.write` call before `_maybe_rotate`, so a batch never spans two WAL files under normal operation
- Source: entries/2026/05/29/topic-wal-recovery-semantics.md

### [REJECT] wal-tail-corruption-safe
Covered by existing `wal-corruption-returns-valid-prefix` and `wal-partial-read-is-eof` which together establish that partial/torn writes at EOF are handled safely
- Source: entries/2026/05/29/topic-wal-recovery-semantics.md

### [ACCEPT] btree-wal-is-physical-redo-only
The B-tree WAL in `btree.py` stores full after-image page data (physical WAL) and supports only redo recovery; there is no undo capability
- Source: entries/2026/05/29/topic-write-ahead-logging-fsync-ordering.md

### [REJECT] commit-means-data-sync-then-wal-truncate
Duplicates existing `wal-commit-sync-before-truncate` which already captures the ordering invariant that data is fsynced before the WAL is truncated
- Source: entries/2026/05/29/topic-write-ahead-logging-fsync-ordering.md

### [ACCEPT] btree-wal-replay-is-idempotent
Because the B-tree WAL stores complete page images, replaying entries already applied to the data file produces identical results, making the crash window after `sync()` but before `truncate()` harmless
- Source: entries/2026/05/29/topic-write-ahead-logging-fsync-ordering.md

### [ACCEPT] two-wal-designs-in-repo
The standalone `wal.py` uses a logical WAL (keyed PUT/DELETE operations with COMMIT markers) while `btree.py` uses a physical WAL (raw page images); they solve different problems and have different recovery semantics
- Source: entries/2026/05/29/topic-write-ahead-logging-fsync-ordering.md

### [ACCEPT] no-atomic-file-creation
No SSTable writer or compaction routine in any implementation uses the write-temp/fsync/rename pattern; all write directly to the final file path
- Source: entries/2026/05/29/topic-write-to-temp-then-rename-pattern.md

### [REJECT] compaction-deletes-before-fsync
Covered by existing `lsm-compaction-deletes-without-fsync-barrier` which captures the same missing fsync barrier between output write and input deletion
- Source: entries/2026/05/29/topic-write-to-temp-then-rename-pattern.md

### [ACCEPT] fsync-used-only-for-appends
`os.fsync` appears in the WAL append path and Bitcask record writes but never in SSTable creation, compaction output, or hint file writes — durability is applied to the append hot path only
- Source: entries/2026/05/29/topic-write-to-temp-then-rename-pattern.md

### [ACCEPT] unbundled-wal-truncate-memory-only
`WriteAheadLog.truncate_before` in the unbundled database removes entries from memory but does not modify the on-disk WAL file, so reloading from `persist_path` restores truncated entries
- Source: entries/2026/05/29/unbundled-database-unbundled_database-truncate_before.md

### [ACCEPT] unbundled-wal-truncate-preserves-lsn
Truncation in the unbundled database WAL never resets `_next_lsn`, so appending after truncation continues with monotonically increasing LSNs
- Source: entries/2026/05/29/unbundled-database-unbundled_database-truncate_before.md

### [ACCEPT] unbundled-wal-truncate-keeps-gte
The unbundled database WAL cutoff is exclusive-below: entries with `lsn >= cutoff` are retained, entries with `lsn < cutoff` are discarded
- Source: entries/2026/05/29/unbundled-database-unbundled_database-truncate_before.md

### [ACCEPT] unbundled-wal-entries-ordered-by-lsn
WAL entries in the unbundled database's `_entries` list are always in ascending LSN order, maintained by the sequential nature of `append()`
- Source: entries/2026/05/29/unbundled-database-unbundled_database-truncate_before.md

### [ACCEPT] wal-eof-advances-to-next-file
An EOF or partial read within a single WAL file causes `_read_all_records` to advance to the next file rather than terminate the entire iterator
- Source: entries/2026/05/29/write-ahead-log-wal-_read_all_records.md

### [ACCEPT] wal-recover-seq-vs-read-all-differ
`_recover_seq_num` uses `break` on corruption (continues to next file) while `_read_all_records` uses `return` (stops everything) — recovery is lenient to maximize the sequence counter, replay is strict to avoid replaying past corruption
- Source: entries/2026/05/29/write-ahead-log-wal-_read_all_records.md

### [REJECT] wal-corruption-stops-all-files
Duplicates existing `wal-crc-mismatch-halts-all-replay` which already captures that CRC mismatch terminates the entire replay/iteration
- Source: entries/2026/05/29/write-ahead-log-wal-_read_all_records.md

### [REJECT] wal-read-all-requires-caller-flush
Covered by existing `replay-flush-before-read` (replay flushes before reading) and `iterate-flushes-without-fsync` (iterate flushes without fsync)
- Source: entries/2026/05/29/write-ahead-log-wal-_read_all_records.md

### [REJECT] wal-read-all-no-lock
Covered by existing `replay-not-lock-protected-during-read` which captures the same locking gap
- Source: entries/2026/05/29/write-ahead-log-wal-_read_all_records.md


### [ACCEPT] recover-seq-num-is-global-max
`_recover_seq_num` returns the maximum sequence number across all WAL files, not just the latest file, ensuring correctness even when records are spread across rotated segments.
- Source: entries/2026/05/29/write-ahead-log-wal-_recover_seq_num.md

### [ACCEPT] recover-seq-skips-corrupt-files
`_recover_seq_num` treats CRC errors as file-local (abandons the corrupted file, continues scanning subsequent files), unlike `_read_all_records` which stops all scanning on corruption.
- Source: entries/2026/05/29/write-ahead-log-wal-_recover_seq_num.md

### [REJECT] seq-num-zero-means-empty
Derivative of existing belief `wal-seq-starts-at-one` — if seq starts at 1, a recovery result of 0 trivially means no records exist.
- Source: entries/2026/05/29/write-ahead-log-wal-_recover_seq_num.md

### [REJECT] recover-is-read-only
Trivially expected of a recovery scan function; this is not an invariant that could be accidentally violated or that constrains design decisions.
- Source: entries/2026/05/29/write-ahead-log-wal-_recover_seq_num.md

### [ACCEPT] recover-seq-no-gap-detection
`_recover_seq_num` finds the max sequence number but does not detect or report gaps in the sequence space, meaning lost records between rotations or corruption are silently ignored.
- Source: entries/2026/05/29/write-ahead-log-wal-_recover_seq_num.md

### [ACCEPT] wal-rotate-fsync-before-close
`_rotate` always fsyncs the outgoing segment before closing it, ensuring no buffered writes are lost during segment rotation.
- Source: entries/2026/05/29/write-ahead-log-wal-_rotate.md

### [ACCEPT] wal-segment-naming-zero-padded
WAL segment filenames are zero-padded 6-digit integers (`000001.wal`, `000002.wal`, ...) derived from the highest existing filename plus one, which makes lexicographic sort equal numeric sort.
- Source: entries/2026/05/29/write-ahead-log-wal-_rotate.md

### [ACCEPT] wal-single-writer-fd
At most one file descriptor is open for WAL writes at any time; `_rotate` closes the old fd before opening the new one.
- Source: entries/2026/05/29/write-ahead-log-wal-_rotate.md

### [ACCEPT] wal-rotate-no-gap-protection
If a segment file is manually deleted from the directory, `_rotate` can produce a filename that reuses a previously-used number, since it derives the next name solely from the current highest filename.
- Source: entries/2026/05/29/write-ahead-log-wal-_rotate.md



---

**Generated:** 2026-05-29
**Source:** 116 entries from entries/
**Model:** claude

### [ACCEPT] avro-no-self-describing-format
Avro binary encoding contains no type tags, field names, or schema metadata; decoding is impossible without the writer schema.
- Source: entries/2026/05/29/avro-serializer-avro_serializer.md

### [ACCEPT] avro-writer-reader-dual-schema-resolution
AvroDecoder always requires both a writer schema (to parse the wire bytes) and a reader schema (to shape the output); this dual-schema resolution is Avro's core mechanism for schema evolution.
- Source: entries/2026/05/29/avro-serializer-avro_serializer.md

### [ACCEPT] avro-promotions-one-directional
Type promotion during schema resolution is strictly one-directional (int→long→float→double, int→float, int→double, long→double); reverse promotion (e.g., long→int) is not supported and raises SchemaCompatibilityError.
- Source: entries/2026/05/29/avro-serializer-avro_serializer.md

### [ACCEPT] avro-record-fields-wire-order
Record fields are encoded and decoded in declaration order of the writer schema with no field names on the wire; reordering fields in the schema definition changes the binary format and breaks existing data.
- Source: entries/2026/05/29/avro-serializer-avro_serializer.md

### [ACCEPT] avro-missing-reader-field-requires-default
During record resolution, a reader field absent from the writer schema must have a default value; missing defaults cause SchemaCompatibilityError, making defaults the mechanism for forward/backward compatibility.
- Source: entries/2026/05/29/avro-serializer-avro_serializer.md

### [ACCEPT] avro-union-no-duplicate-types
A union schema cannot contain two branches with the same type name (or record name); this constraint is enforced at schema parse time in Schema._parse.
- Source: entries/2026/05/29/avro-serializer-avro_serializer.md

### [ACCEPT] avro-schema-error-vs-compatibility-error
SchemaError is raised at parse time for structurally invalid definitions (a valid Schema object is guaranteed well-formed); SchemaCompatibilityError is raised during decode resolution or compatibility checking when two schemas cannot be reconciled.
- Source: entries/2026/05/29/avro-serializer-avro_serializer.md

### [ACCEPT] avro-no-default-sentinel
_NO_DEFAULT is a sentinel object distinguishing "no default provided" from a None/null default; this matters because null is a valid Avro default value, and conflating the two would break canonical form computation and default-filling during resolution.
- Source: entries/2026/05/29/avro-serializer-avro_serializer.md

### [ACCEPT] avro-registry-4byte-schema-id
SchemaRegistry prepends a 4-byte big-endian unsigned int schema ID (struct.pack('>I', id)) to encoded payloads, modeling Confluent's Schema Registry wire format; schema IDs are limited to [0, 2^32).
- Source: entries/2026/05/29/avro-serializer-avro_serializer.md

### [REJECT] iter-yields-sorted-pairs
Covered by existing beliefs `btree-leaf-sibling-chain` and `range-scan-follows-sibling-chain`; __iter__ uses the same leaf-chain mechanism.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-__iter__.md

### [ACCEPT] iter-no-counter-reset
Unlike get, put, delete, and range_scan, BTree.__iter__ does not call reset_counters() before running, so I/O page-read stats accumulate on top of any prior count — a gotcha when benchmarking iteration.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-__iter__.md

### [REJECT] leaf-chain-invariant
Substantially covered by existing belief `btree-leaf-sibling-chain`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-__iter__.md

### [REJECT] iter-not-mutation-safe
Duplicate of existing belief `iterate-not-snapshot-isolated`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-__iter__.md

### [REJECT] leaf-wire-format-is-header-sibling-then-kv-pairs
Duplicate of existing belief `leaf-wire-format-is-header-sibling-entries`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_deserialize_leaf.md

### [ACCEPT] btree-keys-decoded-as-utf8
Leaf deserialization always decodes keys as UTF-8 strings while values remain raw bytes; invalid UTF-8 in key bytes causes UnicodeDecodeError at read time, not write time (since serialize accepts both str and bytes).
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_deserialize_leaf.md

### [REJECT] max-key-or-value-size-is-65535-bytes
Key side covered by existing belief `key-length-cap-uint16`; value side uses the same uint16 length prefix but page size is the practical limit.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_deserialize_leaf.md

### [ACCEPT] btree-deserialize-no-validation
Both _deserialize_leaf and _deserialize_internal perform no validation of page type, sort order, or buffer length; they trust that WAL CRC checks and the serialize functions guarantee well-formed pages, so corruption produces silent wrong results rather than errors.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_deserialize_leaf.md

### [REJECT] sibling-pointer-enables-sequential-scan-without-parent-traversal
Covered by existing beliefs `btree-leaf-sibling-chain` and `range-scan-follows-sibling-chain`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_deserialize_leaf.md

### [REJECT] find-leaf-read-count
Covered by existing belief `btree-lookup-reads-bounded-by-height`; _find_leaf has the same height-proportional I/O cost.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_find_leaf.md

### [ACCEPT] find-leaf-bisect-right-consistency
_find_leaf, _search, and _insert all use bisect_right (not bisect_left) for child-pointer routing, ensuring all three methods agree on which leaf owns a given key; this is consistent with the split strategy where the separator key equals the first key of the right sibling.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_find_leaf.md

### [ACCEPT] find-leaf-no-type-check
_find_leaf does not verify that a page is an internal node before calling _deserialize_internal; a corrupted tree height or misidentified page type produces silently wrong results rather than an error.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_find_leaf.md

### [ACCEPT] find-leaf-range-scan-only
_find_leaf is called exclusively by range_scan; point lookups use _search instead, which reads the leaf inline and returns the value rather than the page number.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_find_leaf.md

### [ACCEPT] internal-node-children-invariant
_serialize_internal assumes len(children) == len(keys) + 1 but does not assert it; too few children raises IndexError, while extra children beyond keys+1 are silently dropped — a potential data-loss path if the invariant is violated.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_serialize_internal.md

### [ACCEPT] internal-page-binary-layout
Internal page wire format is [type:1B][num_keys:2B][child0:4B] followed by repeating [keylen:2B][key:varB][child:4B], all big-endian with no alignment padding; this interleaved layout mirrors the logical separator-pointer structure.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_serialize_internal.md

### [REJECT] serialize-internal-is-pure
Trivially analogous to existing belief `serialize-leaf-is-pure`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_serialize_internal.md

### [REJECT] key-length-cap-uint16
Already exists as an accepted belief.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_serialize_internal.md

### [ACCEPT] no-page-size-enforcement-in-serialize
Neither _serialize_internal nor _serialize_leaf checks whether the serialized output fits within page_size; overflow prevention is the caller's responsibility via the max_keys limit in BTree._insert.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_serialize_internal.md


### [ACCEPT] page-padding-to-page-size
`_wal_write_page` pads data to exactly `page_size` with null bytes before passing it to both `wal.log_write` and `pm.write_page`, ensuring the WAL record and data file page are byte-identical.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_wal_write_page.md

### [REJECT] wal-before-data-invariant
Duplicate of existing `btree-wal-before-data`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_wal_write_page.md

### [REJECT] wal-commit-required
The mechanics are covered by `wal-commit-clears-all-entries` and `wal-commit-sync-before-truncate`; caller responsibility is implied by `btree-insert-no-wal-commit`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_wal_write_page.md

### [ACCEPT] no-direct-page-writes
The BTree class never calls `pm.write_page` directly for data pages; all data page writes are routed through `_wal_write_page` (or `_wal_write_meta` for page 0) to maintain the write-ahead invariant.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_wal_write_page.md

### [REJECT] btree-delete-no-rebalance
Duplicate of existing `btree-no-underflow-rebalancing` and `btree-delete-cleanup-depth2-only`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-delete.md

### [REJECT] btree-delete-wal-gap
Duplicate of existing `btree-free-page-bypasses-wal`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-delete.md

### [ACCEPT] btree-delete-metadata-reread
`delete` re-reads metadata after `_delete` returns to pick up `next_free`/`free_head` changes made by `pm.free_page`, rather than threading updated values through the recursive call stack.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-delete.md

### [REJECT] btree-delete-bool-coercion
Too implementation-specific; the `'empty'` sentinel coercion via `bool(found)` is a trivial return-value detail, not an architectural invariant.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-delete.md

### [ACCEPT] btree-delete-no-commit-on-miss
When the key is not found, `delete` writes no WAL entries and performs no commit, making failed deletes purely read-only operations.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-delete.md

### [ACCEPT] free-list-is-lifo
The page free list is LIFO: `free_page` pushes to the head and `allocate_page` pops from the head, so the most recently freed page is reused first.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-free_page.md

### [REJECT] free-page-not-wal-protected
Duplicate of existing `btree-free-page-bypasses-wal` and `btree-delete-leaks-pages`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-free_page.md

### [REJECT] free-list-intrusive-storage
Duplicate of existing `btree-free-list-intrusive`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-free_page.md

### [ACCEPT] no-double-free-detection
Calling `free_page` twice on the same page creates a cycle in the free list, which `allocate_page` cannot detect, leading to silent data corruption through repeated page reuse.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-free_page.md

### [ACCEPT] free-page-only-called-from-delete
`PageManager.free_page` is only invoked by `BTree._delete` when removing an emptied leaf node; no other code path frees pages.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-free_page.md

### [REJECT] wal-append-only-until-commit
Duplicate of existing `wal-commit-clears-all-entries` and `wal-is-redo-only`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-log_write.md

### [REJECT] wal-checksum-covers-data-only
Duplicate of existing `all-three-implementations-use-payload-only-crc` and `wal-crc-does-not-cover-seqnum`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-log_write.md

### [ACCEPT] wal-fsync-per-entry
Each `log_write` call forces an `os.fsync`, guaranteeing per-entry durability at the cost of one sync syscall per logged page write.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-log_write.md

### [REJECT] wal-seq-monotonic-per-session
Duplicate of existing `wal-seq-reset-on-commit` and `wal-seq-starts-at-one`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-log_write.md

### [REJECT] wal-single-writer-assumed
Duplicate of existing `wal-single-writer-fd` and `wal-provides-crash-safety-not-concurrency`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-log_write.md

### [REJECT] btree-put-wal-atomicity
Covered by existing `btree-wal-before-data`, `btree-wal-provides-split-atomicity`, and `btree-wal-replay-is-idempotent`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-put.md

### [REJECT] btree-put-three-outcomes
Duplicate of existing `btree-recursive-insert-returns-union`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-put.md

### [ACCEPT] btree-single-writer
`put` (and `delete`) assume single-threaded access: metadata is re-read after `_insert`/`_delete` without any locking or compare-and-swap, creating a TOCTOU window under concurrent writers.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-put.md

### [REJECT] btree-root-split-grows-height
Substantially covered by existing `tree-height-monotonically-increases`.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-put.md

### [ACCEPT] btree-put-metadata-reread
`put` must re-read metadata after `_insert` returns because `PageManager.allocate_page` mutates `next_free` and `free_head` as a side effect during splits, and those changes are not threaded back through the return value.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-put.md


### [REJECT] wal-recover-checksum-gate
Duplicate of existing `btree-crc32-skips-corrupt-entries`
- Source: entries/2026/05/29/b-tree-storage-engine-btree-recover.md

### [ACCEPT] wal-recover-then-truncate
WAL recovery fsyncs the data file via `page_manager.sync()` before truncating the WAL, so a crash during recovery itself leaves the WAL intact for re-replay
- Source: entries/2026/05/29/b-tree-storage-engine-btree-recover.md

### [REJECT] wal-recover-idempotent
Duplicate of existing `btree-wal-replay-is-idempotent`
- Source: entries/2026/05/29/b-tree-storage-engine-btree-recover.md

### [ACCEPT] wal-seq-unused-in-recovery
The 4-byte sequence number field in each WAL entry is parsed during `recover()` but never consulted; replay order is determined solely by file offset
- Source: entries/2026/05/29/b-tree-storage-engine-btree-recover.md

### [ACCEPT] wal-recover-on-every-startup
`BTree.__init__` calls `WAL.recover()` unconditionally on every startup; an empty WAL short-circuits after a single read with no further I/O
- Source: entries/2026/05/29/b-tree-storage-engine-btree-recover.md

### [ACCEPT] pipeline-lazy-pull-model
Pipeline execution is pull-based via nested Python generators; records flow only when the terminal consumer iterates, and stages interleave execution without threads
- Source: entries/2026/05/29/batch-word-count-pipeline.md

### [ACCEPT] sort-external-merge
`Sort` stage spills sorted chunks to temp JSONL files when the in-memory buffer exceeds `memory_limit`, then uses `heapq.merge` with a `KeyedRecord` wrapper (seq tiebreaker) for stable k-way merge
- Source: entries/2026/05/29/batch-word-count-pipeline.md

### [ACCEPT] count-is-barrier
`Count` stage materializes its entire input into a `defaultdict` before yielding any output, making it a pipeline barrier that forces all upstream stages to complete first
- Source: entries/2026/05/29/batch-word-count-pipeline.md

### [ACCEPT] stage-timing-subtracts-downstream
`Stage._tracked_process` subtracts time spent after each `yield` (downstream processing time) to attribute wall-clock time accurately to each individual stage
- Source: entries/2026/05/29/batch-word-count-pipeline.md

### [ACCEPT] no-schema-validation
The batch pipeline performs no validation that adjacent stages produce/consume compatible record formats; mismatched tuple shapes fail at runtime with indexing errors
- Source: entries/2026/05/29/batch-word-count-pipeline.md

### [ACCEPT] cbf-saturated-counters-never-decrement
When a `CountingBloomFilter` counter reaches `_max_val` (default 15 for 4-bit counters), it is permanently frozen: neither `add` nor `remove` will change it
- Source: entries/2026/05/29/bloom-filter-bloom_filter-CountingBloomFilter.md

### [ACCEPT] cbf-remove-false-accept
`CountingBloomFilter.remove` can silently succeed for an item never added if all its hash positions have non-zero counters from other items, corrupting the filter state
- Source: entries/2026/05/29/bloom-filter-bloom_filter-CountingBloomFilter.md

### [ACCEPT] cbf-one-byte-per-counter
Each `CountingBloomFilter` counter uses a full byte in a `bytearray` regardless of `counter_bits`, using 8x more memory than a bit-packed representation
- Source: entries/2026/05/29/bloom-filter-bloom_filter-CountingBloomFilter.md

### [ACCEPT] cbf-count-tracks-calls-not-cardinality
`CountingBloomFilter.__len__` returns `add` calls minus `remove` calls, not distinct element count; adding the same item twice and removing once leaves `len == 1`
- Source: entries/2026/05/29/bloom-filter-bloom_filter-CountingBloomFilter.md

### [ACCEPT] crdt-merge-is-idempotent
All four CRDT `merge` methods (`GCounter`, `PNCounter`, `LWWRegister`, `ORSet`) are idempotent: merging the same state twice produces the same result as merging once
- Source: entries/2026/05/29/conflict-free-replicated-data-types-crdts.md

### [ACCEPT] pncounter-composes-gcounters
`PNCounter` delegates entirely to two `GCounter` instances (`p` for increments, `n` for decrements); it contains no independent merge or counting logic
- Source: entries/2026/05/29/conflict-free-replicated-data-types-crdts.md

### [ACCEPT] orset-add-wins-semantics
Concurrent add and remove of the same element in `ORSet` resolves in favor of the add, because the add's unique tag was not in the remover's tombstone snapshot
- Source: entries/2026/05/29/conflict-free-replicated-data-types-crdts.md

### [ACCEPT] lww-tiebreak-is-deterministic
`LWWRegister.merge` uses `(timestamp, writer_id)` tuple comparison, making conflict resolution a total order with no ambiguity since replica IDs are unique
- Source: entries/2026/05/29/conflict-free-replicated-data-types-crdts.md

### [ACCEPT] sync-all-two-rounds-suffices
`CRDTReplicaGroup.sync_all` runs exactly 2 rounds of all-pairs sync, which is sufficient for convergence of any state-based CRDT with a commutative/associative/idempotent merge
- Source: entries/2026/05/29/conflict-free-replicated-data-types-crdts.md

### [ACCEPT] ch-ring-sorted-invariant
`ConsistentHashRing._ring_positions` is maintained in sorted order at all times via `bisect`; all key lookups depend on this invariant
- Source: entries/2026/05/29/consistent-hashing-consistent_hashing.md

### [ACCEPT] ch-vnode-count-is-weighted
A consistent hash ring node with weight `w` gets `int(num_vnodes * w)` virtual nodes, so weight=0.5 produces half the default ring presence
- Source: entries/2026/05/29/consistent-hashing-consistent_hashing.md

### [ACCEPT] ch-preference-list-skips-duplicates
`get_nodes` walks clockwise past virtual nodes belonging to already-seen physical nodes, ensuring exactly `replication_factor` distinct physical nodes in the preference list
- Source: entries/2026/05/29/consistent-hashing-consistent_hashing.md

### [ACCEPT] ch-transfer-maps-are-descriptive
`add_node` and `remove_node` return transfer map descriptions of which arcs move between nodes but do not move data themselves; the caller must act on the map
- Source: entries/2026/05/29/consistent-hashing-consistent_hashing.md

### [ACCEPT] ch-md5-masked-to-32-bits
The consistent hash ring's `_hash` function truncates MD5's 128-bit output to 32 bits via `& 0xFFFFFFFF`, matching the `RING_SIZE` of 2^32
- Source: entries/2026/05/29/consistent-hashing-consistent_hashing.md


### [ACCEPT] reconstruct-state-is-pure-read
`reconstruct_state` never modifies the `EventStore`; it is a read-only fold over a single stream's events that returns a locally-constructed dict
- Source: entries/2026/05/29/event-sourcing-store-event_store-reconstruct_state.md

### [ACCEPT] reconstruct-state-up-to-is-global-id
The `up_to` parameter in `reconstruct_state` filters on global `event_id`, not stream-local version number, despite operating on a single stream
- Source: entries/2026/05/29/event-sourcing-store-event_store-reconstruct_state.md

### [ACCEPT] reconstruct-state-no-snapshot-support
`reconstruct_state` always replays from the beginning of the stream via `read_stream` with default `from_version=0`; it does not leverage snapshots, unlike `Projection`
- Source: entries/2026/05/29/event-sourcing-store-event_store-reconstruct_state.md

### [ACCEPT] handlers-must-mutate-state-in-place
Both `reconstruct_state` and `Projection` expect handler functions to mutate the `state` dict in place rather than returning a new value; a handler that returns without mutating silently loses its changes
- Source: entries/2026/05/29/event-sourcing-store-event_store-reconstruct_state.md

### [ACCEPT] unhandled-event-types-silently-skipped
Events whose `event_type` has no entry in the handlers dict are skipped without warning in both `reconstruct_state` and `Projection.catch_up`
- Source: entries/2026/05/29/event-sourcing-store-event_store-reconstruct_state.md

### [ACCEPT] event-ids-are-1-based-sequential
`event_id` is a global sequence number assigned as `len(self._events) + 1`, starting at 1 and incrementing by 1 for each appended event with no gaps within a process lifetime
- Source: entries/2026/05/29/event-sourcing-store-event_store.md

### [ACCEPT] stream-version-is-event-count
`stream_version()` returns the count of events in a stream (not the latest `event_id`), and `expected_version` checks against this count for optimistic concurrency
- Source: entries/2026/05/29/event-sourcing-store-event_store.md

### [ACCEPT] batch-append-not-crash-safe
`append_batch` can leave a partial batch on disk if the process crashes mid-write, since events are individually appended to the NDJSON file without a transaction marker or write-ahead log
- Source: entries/2026/05/29/event-sourcing-store-event_store.md

### [ACCEPT] live-projection-is-synchronous
`LiveProjection` processes events synchronously inside the `append()` call path via the `_subscribers` mechanism, meaning the caller blocks until all live projections have updated
- Source: entries/2026/05/29/event-sourcing-store-event_store.md

### [ACCEPT] snapshots-are-in-memory-only
Projection snapshots are stored in `store._snapshots` (an in-memory dict), not persisted to disk, so they are lost on process restart
- Source: entries/2026/05/29/event-sourcing-store-event_store.md

### [ACCEPT] es-verify-is-script-not-pytest
`test_verify.py` runs as a standalone Python script with inline assertions and a final print statement, not through a test framework like pytest
- Source: entries/2026/05/29/event-sourcing-store-test_verify.md

### [REJECT] es-projection-position-tracks-global-offset
Duplicates existing belief `es-global-position-tracks-all-streams` which already covers that projection position is a global offset across all streams
- Source: entries/2026/05/29/event-sourcing-store-test_verify.md

### [REJECT] es-expected-version-is-optimistic-lock
Duplicates existing belief `es-optimistic-concurrency-via-expected-version`
- Source: entries/2026/05/29/event-sourcing-store-test_verify.md

### [REJECT] es-snapshot-preserves-position-and-state
Duplicates existing belief `es-snapshot-saves-state-and-position`
- Source: entries/2026/05/29/event-sourcing-store-test_verify.md

### [ACCEPT] fencing-token-monotonic
`LockService._counter` starts at 1, increments by 1 on every successful `acquire()`, and is never decremented or reset, guaranteeing strict monotonicity of issued tokens
- Source: entries/2026/05/29/fencing-tokens-fencing_tokens.md

### [ACCEPT] fencing-rejects-stale-writes
`FencedResourceServer.write()` rejects any write with a fencing token strictly less than the highest token previously seen for that resource; equal tokens are accepted
- Source: entries/2026/05/29/fencing-tokens-fencing_tokens.md

### [ACCEPT] renew-preserves-token-number
`LockService.renew()` extends the lock TTL by mutating `issued_at` and `ttl` without changing the `FencingToken.token` value, while re-`acquire()` by the same client issues a new higher token
- Source: entries/2026/05/29/fencing-tokens-fencing_tokens.md

### [ACCEPT] fencing-no-exceptions-on-denial
All denial conditions in fencing tokens (lock held, stale token, wrong client) return sentinel values (`None`, `False`, or error dicts) rather than raising exceptions, modeling distributed system responses
- Source: entries/2026/05/29/fencing-tokens-fencing_tokens.md

### [ACCEPT] unfenced-server-accepts-all
`UnfencedResourceServer` has no token parameter on `write()` and accepts all writes unconditionally, serving as the pedagogical unsafe baseline for comparison with `FencedResourceServer`
- Source: entries/2026/05/29/fencing-tokens-fencing_tokens.md

### [ACCEPT] gossip-merge-uses-heartbeat-monotonicity
`receive_gossip` only accepts remote state when the remote heartbeat counter strictly exceeds the local counter, except for death notifications at equal counters which are also accepted
- Source: entries/2026/05/29/gossip-protocol-gossip_protocol.md

### [ACCEPT] gossip-node-status-lifecycle
Node status follows `alive → suspected → dead → removed` with configurable timeouts `t_suspect`, `t_dead`, `t_cleanup` governing transitions; a suspected node can return to alive if a higher heartbeat counter arrives
- Source: entries/2026/05/29/gossip-protocol-gossip_protocol.md

### [ACCEPT] gossip-deep-copy-isolation
All inter-node data transfer (`send_gossip`, `join`, `get_membership_list`) uses `copy.deepcopy` to prevent shared mutable state between simulated nodes
- Source: entries/2026/05/29/gossip-protocol-gossip_protocol.md

### [ACCEPT] gossip-voluntary-leave-broadcasts-to-all
A leaving node sends its death status to every active peer (not just a random one) before deactivating, unlike normal gossip which uses random pairwise exchange
- Source: entries/2026/05/29/gossip-protocol-gossip_protocol.md

### [ACCEPT] gossip-dead-nodes-filtered-on-receive
When receiving gossip, unknown nodes that arrive with `dead` status are silently dropped to prevent zombie membership entries from propagating through the cluster
- Source: entries/2026/05/29/gossip-protocol-gossip_protocol.md


### [ACCEPT] hash-index-write-no-index-mutation
`_write_record` in `hash-index-storage/bitcask.py` does not modify `keydir`; the caller (`put` or `delete`) is responsible for updating the in-memory index after the write returns.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_write_record.md

### [REJECT] write-record-is-append-only
Duplicates existing belief `put-append-only`.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_write_record.md

### [REJECT] tombstone-convention-empty-value
Duplicates existing belief `bitcask-tombstone-is-empty-string`.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_write_record.md

### [ACCEPT] hash-index-bitcask-no-checksum
The hash-index-storage Bitcask records contain no CRC or checksum field, unlike the log-structured-hash-table implementation; on-disk corruption is undetectable during reads or compaction.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_write_record.md

### [ACCEPT] hash-index-sync-writes-controls-fsync
In hash-index-storage Bitcask, the `sync_writes` flag controls whether each `_write_record` call fsyncs to disk; when `False`, recently written records may be lost on crash.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_write_record.md

### [ACCEPT] hash-index-wall-clock-not-monotonic
`_write_record` timestamps use `time.time()`, which is not monotonic — NTP adjustments or manual clock changes can cause a newer write to carry an older timestamp, confusing compaction's latest-wins conflict resolution.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_write_record.md

### [REJECT] no-crc-on-records
Subsumed by the more specific `hash-index-bitcask-no-checksum` proposed above.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_write_record.md

### [REJECT] fsync-controlled-by-sync-writes
Subsumed by `hash-index-sync-writes-controls-fsync` proposed above.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_write_record.md

---

### [ACCEPT] hinted-handoff-one-hint-per-target-per-write
Each unavailable preferred replica gets at most one hint stored on one non-preferred node per write operation, preventing hint explosion.
- Source: entries/2026/05/29/hinted-handoff-hinted_handoff.md

### [ACCEPT] hinted-handoff-reads-skip-non-preferred
`get()` only queries preferred replicas, so data existing only as hints on non-preferred nodes is invisible to reads until handoff delivers it.
- Source: entries/2026/05/29/hinted-handoff-hinted_handoff.md

### [ACCEPT] hinted-handoff-sloppy-quorum-counts-hints
When `sloppy_quorum=True` (the default), stored hints count toward `write_quorum` the same as direct replica writes, trading consistency for write availability.
- Source: entries/2026/05/29/hinted-handoff-hinted_handoff.md

### [ACCEPT] hinted-handoff-version-monotonic-per-key
The coordinator assigns strictly increasing versions per key via a centralized counter, so stale hint replays are absorbed harmlessly by `Node.put()`'s `version >= existing_version` check.
- Source: entries/2026/05/29/hinted-handoff-hinted_handoff.md

### [ACCEPT] hinted-handoff-no-exceptions
Write failures are signaled via `success: False` in the return dict; the hinted handoff module never raises exceptions in normal operation.
- Source: entries/2026/05/29/hinted-handoff-hinted_handoff.md

### [ACCEPT] hinted-handoff-time-is-parameter
All time-dependent operations (`put`, `trigger_handoff`, `expire_all_hints`) take `current_time` as an explicit argument rather than calling `time.time()`, making the implementation fully deterministic and testable.
- Source: entries/2026/05/29/hinted-handoff-hinted_handoff.md

### [ACCEPT] hinted-handoff-hint-cleanup-all-or-nothing
`remove_hints_for()` purges all hints for a recovered target node regardless of whether individual hints were successfully delivered or had expired; after `trigger_handoff`, no hints for that node remain on any other node.
- Source: entries/2026/05/29/hinted-handoff-hinted_handoff.md

---

### [ACCEPT] lamport-receive-tick-guarantees-causal-order
`receive_tick` computes `max(local, received) + 1`, ensuring every receive event has a timestamp strictly greater than the corresponding send event, which is the core Lamport clock invariant.
- Source: entries/2026/05/29/lamport-clocks-lamport.md

### [ACCEPT] lamport-happens-before-uses-graph-not-timestamps
`happens_before()` determines causality by BFS over `_parent`/`_cause` edges in the event DAG, not by comparing timestamps; the `all_events` parameter is accepted but never used.
- Source: entries/2026/05/29/lamport-clocks-lamport.md

### [ACCEPT] lamport-mutex-requires-all-acks
`can_enter()` returns `True` only when the node's request is at the queue head (lowest `(timestamp, node_id)`) AND acknowledgments have been received from every other node in the system.
- Source: entries/2026/05/29/lamport-clocks-lamport.md

### [ACCEPT] lamport-send-delivers-synchronously
`send_message` calls `to_node.receive_message()` directly in the same call stack, making message delivery instantaneous and deterministic with no async queue or network simulation.
- Source: entries/2026/05/29/lamport-clocks-lamport.md

### [ACCEPT] lamport-total-order-breaks-ties-by-node-id
`total_order()` sorts by `(timestamp, node_id)`, using lexicographic node ID comparison as a deterministic tiebreaker when timestamps are equal.
- Source: entries/2026/05/29/lamport-clocks-lamport.md

### [ACCEPT] lamport-timestamp-order-not-implies-causality
`a.timestamp < b.timestamp` does NOT imply a→b; two concurrent events can have ordered timestamps, which is a fundamental limitation of Lamport clocks that vector clocks address.
- Source: entries/2026/05/29/lamport-clocks-lamport.md

---

### [ACCEPT] bully-highest-id-wins
The Bully Algorithm always elects the highest available node ID as leader; `start_election` only sends ELECTION to higher-ID nodes, and `declare_victory` only fires when no higher node responds.
- Source: entries/2026/05/29/leader-election-leader_election.md

### [ACCEPT] bully-synchronous-delivery
All messages generated within a single `tick()` call are delivered and fully resolved (including cascading responses) in a `while all_messages` loop before the tick returns.
- Source: entries/2026/05/29/leader-election-leader_election.md

### [ACCEPT] bully-split-brain-resolved-post-hoc
Split-brain is detected and resolved by `_resolve_split_brain` in the cluster harness after each tick (forcing lower-ID leaders to re-elect), not prevented by the node-level protocol alone.
- Source: entries/2026/05/29/leader-election-leader_election.md

### [ACCEPT] bully-recovery-triggers-election
When a failed node recovers via `recover_node`, it immediately starts an election, potentially preempting the current leader — this is the defining "bully" behavior.
- Source: entries/2026/05/29/leader-election-leader_election.md

### [REJECT] bully-no-external-dependencies
Subsumed by existing beliefs `ddia-pure-python-stdlib` and `ddia-modules-self-contained` which already capture the repo-wide pattern of zero external dependencies.
- Source: entries/2026/05/29/leader-election-leader_election.md

### [ACCEPT] bully-candidate-half-timeout
Candidates wait `election_timeout // 2` for ALIVE responses before deciding, shorter than the follower's full `election_timeout`, to avoid cascading election delays.
- Source: entries/2026/05/29/leader-election-leader_election.md

---

### [REJECT] hint-file-no-tombstones
Duplicates existing belief `bitcask-hint-files-exclude-tombstones`.
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_load_hint_file.md

### [ACCEPT] hint-offset-is-segment-offset
The `offset` value stored in a hint entry is a byte position in the corresponding `.dat` segment file, not a position within the hint file itself; `get()` uses this offset to seek directly into the segment.
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_load_hint_file.md

### [REJECT] recovery-order-correctness
Duplicates existing belief `bitcask-recovery-order-is-ascending`.
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_load_hint_file.md

### [ACCEPT] hint-corrupt-worse-than-missing
A corrupted hint file silently loads bad offsets into the index with no validation; a missing hint file triggers a CRC-checked full scan via `_scan_segment`, making hint corruption strictly worse than hint absence.
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_load_hint_file.md

### [ACCEPT] log-structured-hint-offset-32bit-cap
Hint entry format uses `!II` (two `u32` network-order integers), capping segment file offsets at 4 GiB regardless of `max_segment_size` configuration.
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_load_hint_file.md


### [ACCEPT] bitcask-write-record-no-index-update
`_write_record` only appends to disk and returns a byte offset; it never modifies `self._index`, leaving index management entirely to callers (`put`, `delete`, `compact`)
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_write_record.md

### [REJECT] crc-covers-payload-not-header
Duplicate of existing `bitcask-crc-covers-payload-only` and `all-three-implementations-use-payload-only-crc`
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_write_record.md

### [REJECT] flush-not-fsync
Duplicate of existing `log-structured-bitcask-no-fsync`
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_write_record.md

### [REJECT] record-format-self-describing
Covered by existing `record-format-is-length-prefixed`; the key_size/value_size fields making records self-describing is the direct consequence of length-prefixing
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_write_record.md

### [ACCEPT] bitcask-compact-bypasses-write-record
`compact()` re-serializes records directly to the output file rather than calling `_write_record`, duplicating the binary serialization logic in a separate code path
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_write_record.md

### [ACCEPT] bitcask-get-single-disk-seek
`get()` performs exactly one disk seek per call; the key-to-offset mapping is resolved entirely in memory via `self._index`, making reads O(1) index lookup plus one disk read
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-get.md

### [REJECT] bitcask-get-fresh-handle
Duplicate of existing `bitcask-get-opens-fresh-handle`
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-get.md

### [ACCEPT] bitcask-tombstone-invisible-to-get
`get()` never encounters tombstone records because `delete()` removes the key from `self._index`; tombstone handling is solely a recovery and compaction concern
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-get.md

### [REJECT] bitcask-crc-covers-payload-not-header
Duplicate of existing `bitcask-crc-covers-payload-only`
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-get.md

### [ACCEPT] bitcask-get-no-file-existence-guard
If a segment file is deleted out-of-band or by a crash during compaction, `get()` raises an unhandled `FileNotFoundError` rather than returning `None` or a graceful error
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-get.md

### [REJECT] wal-no-fsync
Duplicate of existing `lsm-wal-has-no-fsync`
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-WAL.md

### [ACCEPT] lsm-wal-crash-tolerant-replay
LSM WAL `replay()` silently discards any trailing partial record by checking remaining bytes at every parse step, making it tolerant of process crashes during `append()`
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-WAL.md

### [ACCEPT] lsm-wal-no-checksums
The LSM WAL format has no checksums or magic bytes; it can detect truncation (incomplete length-prefixed records) but not corruption of existing bytes
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-WAL.md

### [REJECT] wal-truncate-after-flush
Duplicate of existing `lsm-wal-truncated-after-sstable-write`
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-WAL.md

### [REJECT] wal-append-before-memtable
Duplicate of existing `lsm-wal-before-memtable`
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-WAL.md

### [REJECT] wal-truncate-clears-file
Duplicate of existing `lsm-wal-truncate-destroys-immediately`
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-truncate.md

### [ACCEPT] lsm-wal-truncate-wb-then-ab
`WAL.truncate()` uses a two-open sequence — open in `"wb"` mode to truncate the file, then reopen in `"ab"` mode for appending — because `"wb"` positions the cursor at offset 0 which is unsafe for a WAL
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-truncate.md

### [REJECT] wal-truncate-called-after-flush
Duplicate of existing `lsm-wal-truncated-after-sstable-write`
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-truncate.md

### [ACCEPT] lsm-wal-truncate-no-error-recovery
If any `open()` or `close()` call within `WAL.truncate()` fails, the exception propagates uncaught and can leave `self._fd` holding a closed handle, breaking subsequent `append()` calls
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-truncate.md

### [REJECT] wal-no-fsync-truncate
Duplicate of existing `lsm-wal-has-no-fsync`
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-truncate.md

### [ACCEPT] map-side-join-three-strategies
The module implements exactly three join strategies (broadcast hash, partitioned hash, sort-merge) that all produce identical inner-join results when verified by `compare_join_strategies`
- Source: entries/2026/05/29/map-side-join-map_side_joins.md

### [ACCEPT] map-side-join-conflict-prefix
When left and right datasets share a non-key field name, `_merge_records` disambiguates with `left_` and `right_` prefixes; this convention applies in all join types including `None`-fill paths for left joins
- Source: entries/2026/05/29/map-side-join-map_side_joins.md

### [ACCEPT] map-side-join-no-real-parallelism
Mapper parallelism is simulated via round-robin chunking and `_mapper_id` tagging on output records; no threads or processes are used
- Source: entries/2026/05/29/map-side-join-map_side_joins.md

### [ACCEPT] map-side-join-missing-key-skip
Records missing the join key are silently dropped and counted in `stats["skipped_records"]` rather than raising exceptions, across all three join strategies
- Source: entries/2026/05/29/map-side-join-map_side_joins.md

### [ACCEPT] map-side-join-cartesian-on-duplicates
When multiple records share the same join key value, all three strategies produce the cartesian product of the matching left and right groups
- Source: entries/2026/05/29/map-side-join-map_side_joins.md


### [ACCEPT] mr-hash-partitions-keys
MapReduceJob assigns intermediate keys to reducer partitions using `hash(key) % num_reducers`, ensuring all values for a given key reach the same reducer.
- Source: entries/2026/05/29/mapreduce-framework-mapreduce.md

### [ACCEPT] mr-combiner-is-map-side-only
The combiner runs inside `_run_mapper` on each mapper's output for a single partition independently; it never sees data from other mappers or other partitions.
- Source: entries/2026/05/29/mapreduce-framework-mapreduce.md

### [ACCEPT] mr-intermediate-data-uses-json-files
Intermediate shuffle data is serialized as JSON files in a temp directory with naming convention `map-{mapper_id}-part-{partition}.json`.
- Source: entries/2026/05/29/mapreduce-framework-mapreduce.md

### [ACCEPT] mr-fault-tolerant-silently-drops
When `fault_tolerant=True`, mapper and reducer exceptions are caught and silently skipped with no logging or error reporting, producing partial results without any indication of data loss.
- Source: entries/2026/05/29/mapreduce-framework-mapreduce.md

### [ACCEPT] mr-groupby-requires-presort
`_run_reducer` sorts all intermediate pairs by key before calling `itertools.groupby`, which is required because `groupby` only groups consecutive elements with the same key.
- Source: entries/2026/05/29/mapreduce-framework-mapreduce.md

### [ACCEPT] verify-proof-is-pure-static
`verify_proof` is a `@staticmethod` with no side effects; it requires no tree instance, only the data and a `MerkleProof`, so it can run on a different machine than the one that built the tree.
- Source: entries/2026/05/29/merkle-tree-merkle_tree-verify_proof.md

### [ACCEPT] verify-proof-hash-consistency
`verify_proof` concatenates hex-encoded hash strings and encodes to bytes before hashing, matching the exact same scheme used in `__init__` to build internal nodes — a mismatch would silently break all proof verification.
- Source: entries/2026/05/29/merkle-tree-merkle_tree-verify_proof.md

### [ACCEPT] verify-proof-direction-unvalidated
The `direction` field in proof siblings is not validated; any value other than `"left"` is silently treated as `"right"` via the else branch.
- Source: entries/2026/05/29/merkle-tree-merkle_tree-verify_proof.md

### [ACCEPT] verify-proof-root-trust
`verify_proof` confirms data matches a given root hash but does not authenticate the root itself; callers must obtain a trusted root through an independent channel (e.g., a signed block header).
- Source: entries/2026/05/29/merkle-tree-merkle_tree-verify_proof.md

### [ACCEPT] merkle-hashes-over-hex-not-bytes
Internal node hashes are computed over concatenated 64-char hex digest strings (128 ASCII bytes) rather than raw 32-byte SHA-256 digests, resulting in 4x more data per hash input.
- Source: entries/2026/05/29/merkle-tree-merkle_tree-verify_proof.md

### [ACCEPT] ring-topology-needs-multiple-sync-rounds
A 3-node ring topology requires at least 2 `sync()` rounds to achieve full convergence, unlike a fully-connected topology which converges in 1 round.
- Source: entries/2026/05/29/multi-leader-replication-test_multi_leader.md

### [ACCEPT] multi-leader-lamport-monotonic
Successive `put()` calls on the same `ReplicaNode` yield strictly increasing Lamport timestamps.
- Source: entries/2026/05/29/multi-leader-replication-test_multi_leader.md

### [ACCEPT] multi-leader-conflict-log-records-strategy
Conflicts are recorded in `conflict_log` with metadata including the key and the `ConflictStrategy` enum value that resolved them.
- Source: entries/2026/05/29/multi-leader-replication-test_multi_leader.md

### [REJECT] lww-tiebreak-uses-node-id
Duplicates existing belief `multi-leader-lww-deterministic-tiebreak`.
- Source: entries/2026/05/29/multi-leader-replication-test_multi_leader.md

### [REJECT] custom-merge-replaces-lww
Overlaps with existing belief `multi-leader-custom-merge-new-timestamp` which already covers the custom merge strategy behavior.
- Source: entries/2026/05/29/multi-leader-replication-test_multi_leader.md

### [REJECT] sync-is-idempotent
Duplicates existing belief `multi-leader-idempotent-apply`.
- Source: entries/2026/05/29/multi-leader-replication-test_multi_leader.md

### [REJECT] delete-propagates-as-tombstone
Duplicates existing belief `multi-leader-tombstone-delete`.
- Source: entries/2026/05/29/multi-leader-replication-test_multi_leader.md

### [ACCEPT] partitioned-log-offset-monotonic
Offsets within a partition are strictly increasing and never reused; `_base_offsets` only advances forward via `truncate` and `compact_partition`.
- Source: entries/2026/05/29/partitioned-log-partitioned_log.md

### [ACCEPT] partitioned-log-key-routing-deterministic
Messages with the same key always route to the same partition via `MD5(key) % num_partitions`, guaranteeing per-key ordering within a partition.
- Source: entries/2026/05/29/partitioned-log-partitioned_log.md

### [ACCEPT] partitioned-log-compaction-preserves-keyless
Log compaction retains only the last occurrence per key but always preserves messages with `key=None`.
- Source: entries/2026/05/29/partitioned-log-partitioned_log.md

### [ACCEPT] partitioned-log-partition-count-limit
`Topic.__init__` enforces partition count between 1 and 128, raising `ValueError` outside that range.
- Source: entries/2026/05/29/partitioned-log-partitioned_log.md

### [ACCEPT] partitioned-log-rebalance-eager-round-robin
`ConsumerGroup.rebalance()` is synchronous and triggered on every membership change, distributing partitions round-robin across sorted consumer IDs — no incremental or cooperative protocol.
- Source: entries/2026/05/29/partitioned-log-partitioned_log.md

### [ACCEPT] partitioned-log-persistence-jsonl
When `persist_dir` is set, messages are appended as JSONL files named `{topic}_{partition}.jsonl` and committed offsets are written to a single `offsets.json`.
- Source: entries/2026/05/29/partitioned-log-partitioned_log.md

### [ACCEPT] raft-sentinel-log-entry
The log is initialized with a sentinel `LogEntry(term=0, index=0)` at position 0 that is never removed, eliminating empty-log edge cases in `_last_log_index()` and `_last_log_term()`.
- Source: entries/2026/05/29/raft-consensus-raft.md

### [ACCEPT] raft-current-term-commit-only
`_advance_commit_index` only commits entries whose term matches `_current_term`, implementing the Raft safety property from §5.4.2 that prevents committing prior-term entries by replica count alone.
- Source: entries/2026/05/29/raft-consensus-raft.md

### [ACCEPT] raft-single-tick-delivery
`RaftCluster.tick()` collects outbound messages and delivers both the request and its response within the same tick invocation — there is no simulated network latency.
- Source: entries/2026/05/29/raft-consensus-raft.md

### [ACCEPT] raft-no-leader-forwarding
`client_request` returns `{"success": False, "error": "not leader"}` if the node is not the leader; there is no automatic forwarding to the current leader.
- Source: entries/2026/05/29/raft-consensus-raft.md

### [ACCEPT] raft-partition-via-set
Network partitions are simulated by adding node IDs to `_partitioned`; partitioned nodes neither tick nor send/receive messages, modeling complete network isolation.
- Source: entries/2026/05/29/raft-consensus-raft.md


### [ACCEPT] range-partitioning-split-at-median-index
`Partition.split()` divides at `len(keys)//2` (count-based median), not the lexicographic midpoint of the key range, producing equal-count halves but potentially unequal key ranges.
- Source: entries/2026/05/29/range-partitioning-range_partitioning.md

### [ACCEPT] range-partitioning-auto-split-manual-merge
Splits are triggered automatically during `put()` when a partition exceeds `max_partition_size`, but merges require an explicit call to `merge_small_partitions()` and never happen implicitly.
- Source: entries/2026/05/29/range-partitioning-range_partitioning.md

### [ACCEPT] range-partitioning-merge-assumes-adjacency
`Partition.merge(other)` appends the other partition's keys without re-sorting; correctness depends on the caller passing the immediate right neighbor, with no runtime adjacency validation.
- Source: entries/2026/05/29/range-partitioning-range_partitioning.md

### [ACCEPT] range-partitioning-boundary-routing-bisect
Key routing uses `bisect_right(boundaries, key) - 1` on a parallel sorted list of partition start keys, giving O(log p) partition lookup where p is the partition count.
- Source: entries/2026/05/29/range-partitioning-range_partitioning.md

### [ACCEPT] range-partitioning-contiguous-half-open-coverage
Partitions use half-open intervals `[start_key, end_key)` with the first partition starting at `""` and the last having `end_key=None`, guaranteeing the entire string keyspace is covered with no gaps or overlaps.
- Source: entries/2026/05/29/range-partitioning-range_partitioning.md

### [ACCEPT] range-partitioning-no-thread-safety
No concurrency protection exists on any mutation path (`put`, `delete`, `split`, `merge`); concurrent access would corrupt the sorted key/value lists.
- Source: entries/2026/05/29/range-partitioning-range_partitioning.md

### [ACCEPT] range-partitioning-parallel-arrays-routing
`_partitions` and `_boundaries` are kept in lockstep as parallel arrays — index `i` in both refers to the same partition — trading O(n) insert/delete for O(1) indexed access and O(log n) binary search routing.
- Source: entries/2026/05/29/range-partitioning-range_partitioning.md

### [ACCEPT] read-repair-scope-is-quorum-only
Read repair in `get()` only fixes replicas that were part of the read quorum, not all replicas; full-cluster repair requires a separate call to `anti_entropy_repair()`.
- Source: entries/2026/05/29/read-repair-read_repair.md

### [ACCEPT] version-scan-includes-unavailable
`put()` determines the next version by scanning all replicas including those marked unavailable, which prevents version regression but couples version assignment to unavailable node state.
- Source: entries/2026/05/29/read-repair-read_repair.md

### [ACCEPT] replica-put-rejects-older-versions
`Replica.put()` silently rejects writes with a version strictly less than the stored version (returns `False`), enforcing monotonic version advancement; equal versions are accepted as last-writer-wins.
- Source: entries/2026/05/29/read-repair-read_repair.md

### [ACCEPT] quorum-violation-warns-not-raises
When `R + W <= N`, the `ReadRepairStore` constructor emits a `warnings.warn` rather than raising an exception, allowing intentionally relaxed quorum configurations.
- Source: entries/2026/05/29/read-repair-read_repair.md

### [ACCEPT] put-return-value-ignored-by-coordinator
`ReadRepairStore` never checks the boolean return from `Replica.put()` during read repair or writes, relying on version monotonicity to make stale or redundant writes harmless no-ops.
- Source: entries/2026/05/29/read-repair-read_repair.md

### [ACCEPT] doc-partitioned-query-touches-all
`DocumentPartitionedDB.query_by_field` always iterates every partition regardless of result count, touching exactly `num_partitions` partitions (scatter/gather with no short-circuit).
- Source: entries/2026/05/29/secondary-index-partitioning-secondary_index_partitioning.md

### [ACCEPT] term-partitioned-write-touches-multiple
`TermPartitionedDB.put` touches 1 + N partitions where N depends on how many distinct index partitions the document's field values hash to, because index entries live on term-hashed partitions separate from the data partition.
- Source: entries/2026/05/29/secondary-index-partitioning-secondary_index_partitioning.md

### [ACCEPT] term-partitioned-point-query-is-targeted
`TermPartitionedDB.query_by_field` looks up a single index partition by term hash then fetches documents from their home partitions, touching 1 + K partitions (K = distinct data partitions) instead of scattering to all.
- Source: entries/2026/05/29/secondary-index-partitioning-secondary_index_partitioning.md

### [ACCEPT] async-index-defers-mutations
With `async_index=True`, `TermPartitionedDB` queues index operations in `_pending` instead of applying them immediately; `flush_index()` must be called to drain the queue, modeling asynchronous global index updates.
- Source: entries/2026/05/29/secondary-index-partitioning-secondary_index_partitioning.md

### [ACCEPT] secondary-index-stale-entry-cleanup
Both `DocumentPartitionedDB` and `TermPartitionedDB` remove stale index entries before writing new ones during `put` and `delete`, preventing phantom results from old field values.
- Source: entries/2026/05/29/secondary-index-partitioning-secondary_index_partitioning.md

### [ACCEPT] secondary-index-document-class-unused
The `Document` dataclass exists as a domain concept but the DB classes accept `pk` and `fields` separately and never consume `Document` instances.
- Source: entries/2026/05/29/secondary-index-partitioning-secondary_index_partitioning.md

### [ACCEPT] stcs-merges-all-qualifying-buckets-per-call
Size-tiered `run_compaction()` processes every bucket that meets `min_threshold` in a single call, while leveled compaction processes at most one level per call.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-CompactionManager.md

### [ACCEPT] lcs-produces-single-output-sstable
Leveled compaction merges all inputs into one SSTable rather than splitting output by size, which means levels above L0 will contain overlapping key ranges after compaction — violating the real LCS invariant that each level has non-overlapping SSTables.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-CompactionManager.md

### [ACCEPT] compaction-manager-never-deletes-old-files
`CompactionManager` removes compacted SSTables from the in-memory `_sstables` list but never deletes their underlying files on disk; cleanup is the caller's responsibility.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-CompactionManager.md

### [ACCEPT] strategy-selection-is-binary
`CompactionManager` compares `strategy` with `==` against `"size_tiered"` — any other string silently selects leveled compaction with no validation or error.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-CompactionManager.md

### [ACCEPT] lcs-picks-max-overlap-sstable
For level N→N+1 compaction, the SSTable with the most overlapping neighbors in the next level is chosen — the opposite of LevelDB/RocksDB's least-overlap heuristic, increasing write amplification per compaction.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-CompactionManager.md

### [ACCEPT] lcs-compact-one-per-call
`_lcs_compact` performs at most one merge operation per invocation and returns immediately; cascading compactions across levels require the caller to loop on `run_compaction()`.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-_lcs_compact.md

### [ACCEPT] lcs-l0-compacts-all
L0 compaction always includes every L0 SSTable in the merge set (not just a subset), which can cause large write spikes when many L0 SSTables accumulate before the trigger fires.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-_lcs_compact.md

### [REJECT] lcs-picks-most-overlap
Duplicates `lcs-picks-max-overlap-sstable` from the CompactionManager entry — same claim about the same code path.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-_lcs_compact.md

### [REJECT] lcs-no-old-file-cleanup
Duplicates `compaction-manager-never-deletes-old-files` from the CompactionManager entry — same claim about the same class.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-_lcs_compact.md

### [ACCEPT] lcs-level-size-exponential
Level size budgets grow as `base_size × fanout^(level-1)`, so with defaults of 10MB base and 10× fanout: L1=10MB, L2=100MB, L3=1GB, etc.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-_lcs_compact.md


## Stream Join Processor (`stream_join_processor.py`)

### [ACCEPT] stream-join-one-to-many
A single event can match multiple events on the opposite side; the matched flag is set on all participants, so none produce outer-join misses
- Source: entries/2026/05/29/stream-join-processor-stream_join_processor.md

### [ACCEPT] stream-join-miss-deferred
Outer-join misses are only emitted at event expiration time (when timestamp falls below watermark minus window duration), never eagerly on arrival
- Source: entries/2026/05/29/stream-join-processor-stream_join_processor.md

### [ACCEPT] stream-join-left-asymmetric
`JoinType.LEFT` emits misses only for unmatched left-side events; unmatched right-side events are silently dropped without any miss emission
- Source: entries/2026/05/29/stream-join-processor-stream_join_processor.md

### [ACCEPT] stream-join-no-stream-validation
The processor does not validate that an event's `stream_name` matches either configured stream; an unknown stream name is silently treated as the right stream via `_is_left` returning `False`
- Source: entries/2026/05/29/stream-join-processor-stream_join_processor.md

### [ACCEPT] tumbling-window-floor-division
`TumblingWindowAggregator` assigns windows by floor-dividing the timestamp by window size, producing aligned non-overlapping boundaries regardless of when events arrive
- Source: entries/2026/05/29/stream-join-processor-stream_join_processor.md

### [ACCEPT] stream-join-watermark-monotonic
The join processor's watermark only advances forward via `max(current, event.timestamp)`; `advance_time()` silently ignores timestamps at or below the current watermark
- Source: entries/2026/05/29/stream-join-processor-stream_join_processor.md

### [ACCEPT] stream-join-get-results-destructive
`get_results()` swaps out and resets the accumulated results list; callers see only results since the previous drain (pull-based consumption model)
- Source: entries/2026/05/29/stream-join-processor-stream_join_processor.md

## Batch Atomicity Across Rotation (`topic-batch-atomicity-across-rotation.md`)

### [ACCEPT] wal-max-file-size-is-soft
`max_file_size` is a soft limit; a single batch can push a WAL file arbitrarily past it because `_maybe_rotate` runs only after the write and sync complete
- Source: entries/2026/05/29/topic-batch-atomicity-across-rotation.md

### [ACCEPT] wal-rotation-is-post-write
Both `append` and `append_batch` call `_maybe_rotate()` after write and sync complete; rotation never interrupts an in-progress write, and the file only rotates on the next operation
- Source: entries/2026/05/29/topic-batch-atomicity-across-rotation.md

### [REJECT] wal-batch-never-splits
Duplicates existing `wal-batch-single-file`
- Source: entries/2026/05/29/topic-batch-atomicity-across-rotation.md

### [REJECT] wal-batch-forces-fsync
Duplicates existing `wal-batch-always-fsyncs`
- Source: entries/2026/05/29/topic-batch-atomicity-across-rotation.md

### [REJECT] wal-commit-record-is-atomicity-marker
Duplicates existing `wal-batch-commit-sentinel`
- Source: entries/2026/05/29/topic-batch-atomicity-across-rotation.md

## Batch Write Durability (`topic-batch-write-durability.md`)

### [REJECT] wal-append-batch-uses-commit-sentinel
Duplicates existing `wal-batch-commit-sentinel`
- Source: entries/2026/05/29/topic-batch-write-durability.md

### [REJECT] wal-force-true-bypasses-batch-sync
Duplicates existing `force-true-bypasses-sync-mode`
- Source: entries/2026/05/29/topic-batch-write-durability.md

### [REJECT] wal-crc-per-record
Covered by existing `wal-crc-includes-op-type` and `wal-crc-does-not-cover-seqnum`
- Source: entries/2026/05/29/topic-batch-write-durability.md

### [REJECT] wal-append-batch-and-checkpoint-always-fsync
Partially covered by existing `wal-batch-always-fsyncs` and `wal-commit-sync-before-truncate`; also incomplete since `rotation-always-fsyncs` also fsyncs
- Source: entries/2026/05/29/topic-batch-write-durability.md

### [REJECT] wal-atomicity-depends-on-consumer
Duplicates existing `replay-does-not-enforce-batch-atomicity`
- Source: entries/2026/05/29/topic-batch-write-durability.md

## Bitcask Binary Format (`topic-bitcask-binary-format.md`)

### [ACCEPT] bitcask-header-is-12-bytes
The log-structured hash table's `HEADER_FMT = "!III"` produces a fixed 12-byte header (three big-endian uint32s: CRC, key_size, value_size) at `log-structured-hash-table/bitcask.py:10-11`; total record size is always `12 + len(key) + len(value)`
- Source: entries/2026/05/29/topic-bitcask-binary-format.md

### [ACCEPT] bitcask-crc-validated-on-read-only
In the log-structured hash table, CRC32 validation occurs only during `get()` and `_scan_segment()`; writes compute and store the CRC but never verify it back, so a corrupted write is not caught until read time
- Source: entries/2026/05/29/topic-bitcask-binary-format.md

### [REJECT] crc-covers-payload-not-header
Duplicates existing `all-three-implementations-use-payload-only-crc`
- Source: entries/2026/05/29/topic-bitcask-binary-format.md

### [REJECT] record-size-formula
Trivially derived from `bitcask-header-is-12-bytes`
- Source: entries/2026/05/29/topic-bitcask-binary-format.md

### [REJECT] segment-rotation-threshold-depends-on-record-size
Test design detail explaining magic numbers, not an architectural invariant
- Source: entries/2026/05/29/topic-bitcask-binary-format.md

### [REJECT] corruption-detection-on-read-not-write
Subsumed by the more specific `bitcask-crc-validated-on-read-only` above
- Source: entries/2026/05/29/topic-bitcask-binary-format.md

## Bitcask Crash Recovery Guarantees (`topic-bitcask-crash-recovery-guarantees.md`)

### [ACCEPT] sstable-writer-no-fsync-on-close
`SSTableWriter.finish()` at `sstable-and-compaction/sstable.py:91` calls `close()` without prior `fsync()`, leaving newly flushed SSTables vulnerable to loss if the OS page cache hasn't been written to disk
- Source: entries/2026/05/29/topic-bitcask-crash-recovery-guarantees.md

### [ACCEPT] lsm-flush-double-loss-window
The LSM `_flush()` truncates the WAL after writing an SSTable, but since neither the SSTable write nor the truncation is fsynced, a crash can lose both the WAL source data and the SSTable destination simultaneously — making the data irrecoverable
- Source: entries/2026/05/29/topic-bitcask-crash-recovery-guarantees.md

### [REJECT] lsm-wal-no-fsync
Duplicates existing `lsm-wal-has-no-fsync`
- Source: entries/2026/05/29/topic-bitcask-crash-recovery-guarantees.md

### [REJECT] log-hash-table-no-durability-guarantee
Duplicates existing `log-structured-bitcask-no-fsync`
- Source: entries/2026/05/29/topic-bitcask-crash-recovery-guarantees.md

### [REJECT] btree-wal-is-reference-implementation
Comparative judgment across implementations rather than a testable factual claim about a specific module
- Source: entries/2026/05/29/topic-bitcask-crash-recovery-guarantees.md

### [REJECT] bitcask-sync-writes-default-true
Duplicates existing `bitcask-fsync-per-record-default`
- Source: entries/2026/05/29/topic-bitcask-crash-recovery-guarantees.md

### [REJECT] sstable-wal-truncation-race
Subsumed by the more specific `lsm-flush-double-loss-window` above
- Source: entries/2026/05/29/topic-bitcask-crash-recovery-guarantees.md


### [ACCEPT] wal-checkpoint-forces-sync
`checkpoint()` calls `_do_sync(force=True)`, making checkpoint markers always durable on disk regardless of the configured sync mode
- Source: entries/2026/05/29/topic-bitcask-crash-recovery-semantics.md

### [REJECT] wal-break-on-corruption-discards-tail
Duplicates `wal-crc-mismatch-halts-all-replay` and `wal-no-resync-after-corruption`
- Source: entries/2026/05/29/topic-bitcask-crash-recovery-semantics.md

### [REJECT] wal-partial-header-treated-as-eof
Duplicates `wal-partial-read-is-eof`
- Source: entries/2026/05/29/topic-bitcask-crash-recovery-semantics.md

### [REJECT] wal-batch-commit-always-synced
Duplicates `wal-batch-always-fsyncs`
- Source: entries/2026/05/29/topic-bitcask-crash-recovery-semantics.md

### [REJECT] wal-none-mode-no-flush-no-fsync
Duplicates `none-mode-never-syncs-on-append`
- Source: entries/2026/05/29/topic-bitcask-crash-recovery-semantics.md

### [REJECT] wal-crc-covers-content-not-header
Duplicates `all-three-implementations-use-payload-only-crc` combined with `wal-crc-does-not-cover-seqnum`
- Source: entries/2026/05/29/topic-bitcask-crash-recovery-semantics.md

### [ACCEPT] hash-index-hint-self-sufficient
The `hash-index-storage` hint file contains all four keydir fields (file_id, offset, size, timestamp), making it sufficient to fully reconstruct the in-memory index without reading data files
- Source: entries/2026/05/29/topic-bitcask-hint-file-format.md

### [REJECT] log-structured-hint-incomplete
Duplicates `log-structured-index-omits-size-and-timestamp`
- Source: entries/2026/05/29/topic-bitcask-hint-file-format.md

### [REJECT] no-hint-integrity-checking
Duplicates `hint-file-no-integrity-check`
- Source: entries/2026/05/29/topic-bitcask-hint-file-format.md

### [REJECT] hint-files-compaction-only
Duplicates `hint-written-only-during-compaction`
- Source: entries/2026/05/29/topic-bitcask-hint-file-format.md

### [REJECT] log-structured-data-crc-hint-gap
Duplicates `hint-file-no-integrity-check` combined with `bitcask-crc32-raises-corruption-error`
- Source: entries/2026/05/29/topic-bitcask-hint-file-format.md

### [ACCEPT] hash-index-compaction-manual-only
The `hash-index-storage/bitcask.py` `compact()` must be called explicitly by the caller; there is no auto-compact trigger, threshold tracking, or background compaction unlike the `log-structured-hash-table` variant
- Source: entries/2026/05/29/topic-bitcask-paper-comparison.md

### [REJECT] python-bitcask-no-writer-locking
Duplicates `bitcask-single-active-writer` combined with `compact-no-concurrency-safety`
- Source: entries/2026/05/29/topic-bitcask-paper-comparison.md

### [REJECT] keydir-rebuilt-on-every-open
Duplicates `bitcask-keydir-is-sole-index` combined with `bitcask-no-parallel-rebuild`
- Source: entries/2026/05/29/topic-bitcask-paper-comparison.md

### [REJECT] compaction-is-synchronous-and-blocking
Duplicates `compact-single-threaded-assumption`
- Source: entries/2026/05/29/topic-bitcask-paper-comparison.md

### [REJECT] only-log-structured-variant-has-crc
Duplicates `bitcask-no-checksum-validation` (hash-index) and `bitcask-crc32-raises-corruption-error` (log-structured)
- Source: entries/2026/05/29/topic-bitcask-paper-comparison.md

### [REJECT] auto-compact-trigger-uses-segment-count-only
Duplicates `bitcask-auto-compact-threshold`
- Source: entries/2026/05/29/topic-bitcask-paper-comparison.md

### [REJECT] hint-files-are-embarrassingly-parallel
Describes Erlang/Riak's optimization strategy, not a testable property of this Python codebase
- Source: entries/2026/05/29/topic-bitcask-parallel-hint-loading.md

### [REJECT] ets-enables-concurrent-keydir-writes
Describes Erlang ETS internals, not this Python codebase
- Source: entries/2026/05/29/topic-bitcask-parallel-hint-loading.md

### [REJECT] parallel-hint-loading-requires-barrier
Describes Erlang startup protocol, not this Python codebase
- Source: entries/2026/05/29/topic-bitcask-parallel-hint-loading.md

### [REJECT] slow-path-parallelism-requires-ordered-merge
Tombstone ordering during recovery already covered by `bitcask-recovery-order-is-ascending`
- Source: entries/2026/05/29/topic-bitcask-parallel-hint-loading.md

### [ACCEPT] hash-index-keys-must-fit-in-ram
Both Bitcask implementations require every live key to be held in a Python dict in memory; the dataset's key space is bounded by available RAM
- Source: entries/2026/05/29/topic-bitcask-vs-lsm-tree.md

### [ACCEPT] bitcask-no-range-query-support
Neither Bitcask implementation has any range query, ordered iteration, or prefix scan capability; the hash-based keydir only supports exact-key point lookups
- Source: entries/2026/05/29/topic-bitcask-vs-lsm-tree.md

### [REJECT] bitcask-recovery-requires-full-scan-without-hints
Duplicates `bitcask-crash-recovery-without-hints`
- Source: entries/2026/05/29/topic-bitcask-vs-lsm-tree.md

### [REJECT] lsm-wal-limits-recovery-to-unflushed-writes
Duplicates `lsm-wal-truncated-after-sstable-write` (which implies only unflushed data needs recovery)
- Source: entries/2026/05/29/topic-bitcask-vs-lsm-tree.md

### [ACCEPT] hash-index-read-is-single-seek
A Bitcask `get()` does one dict lookup plus one positioned disk read (O(1)), while an LSM-tree `get()` may search the memtable then multiple SSTables from newest to oldest (O(log N) per level)
- Source: entries/2026/05/29/topic-bitcask-vs-lsm-tree.md


### [ACCEPT] no-sibling-sentinel-is-max-uint32
`NO_SIBLING = 0xFFFFFFFF` marks the end of the leaf sibling chain; a leaf with this value is the rightmost at its level
- Source: entries/2026/05/29/topic-blink-tree-paper.md

### [REJECT] leaf-pages-carry-sibling-pointers
Duplicates existing beliefs `btree-leaf-sibling-chain` and `leaf-wire-format-is-header-sibling-entries`
- Source: entries/2026/05/29/topic-blink-tree-paper.md

### [REJECT] no-concurrency-control-exists
Duplicates existing belief `no-concurrency-primitives`
- Source: entries/2026/05/29/topic-blink-tree-paper.md

### [REJECT] sibling-links-serve-range-scans-not-concurrency
Duplicates existing belief `sibling-field-is-structural-only`
- Source: entries/2026/05/29/topic-blink-tree-paper.md

### [ACCEPT] wal-recovery-scans-full-file
`_recover_seq_num` in `write-ahead-log/wal.py` reads every record in every WAL file sequentially from byte zero; there is no seek-to-end or block-skip optimization, making recovery O(file-size)
- Source: entries/2026/05/29/topic-block-aligned-wal-records.md

### [ACCEPT] lsm-wal-has-no-integrity-check
The WAL in `log-structured-merge-tree/lsm.py` uses only length-prefix framing with no CRC or checksum; a single bit flip in the length field causes silent data corruption or misaligned reads
- Source: entries/2026/05/29/topic-block-aligned-wal-records.md

### [ACCEPT] torn-length-prefix-causes-silent-skip
If a torn write corrupts the 4-byte length prefix in `wal.py:_read_record`, the reader interprets garbage as `record_length`, reads that many bytes (consuming valid subsequent records as data), then returns `None` on short read — no error is raised and no resync is attempted
- Source: entries/2026/05/29/topic-block-aligned-wal-records.md

### [ACCEPT] wal-contiguous-no-block-alignment
None of the WAL implementations use block-aligned or page-aligned record layouts; all records are packed contiguously with variable-length encoding, so torn writes can land at arbitrary byte offsets
- Source: entries/2026/05/29/topic-block-aligned-wal-records.md

### [ACCEPT] sstable-sparse-index-in-footer
Both SSTable implementations store the sparse index at the end of the file with a footer pointer, meaning bloom filters could be co-located in the same footer region without changing the file layout
- Source: entries/2026/05/29/topic-bloom-filter-optimization.md

### [REJECT] bloom-filter-uses-double-hashing
Duplicates existing belief `bloom-double-hashing`
- Source: entries/2026/05/29/topic-bloom-filter-optimization.md

### [REJECT] bloom-serialization-exists
Duplicates existing belief `bloom-serialization-footer-ready`
- Source: entries/2026/05/29/topic-bloom-filter-optimization.md

### [ACCEPT] lsm-get-probes-all-sstables-on-miss
`LSMTree.get()` iterates through every SSTable in reverse sequence order and returns only after checking all of them when a key is absent; without bloom filters, missing-key lookups are O(N) in SSTable count
- Source: entries/2026/05/29/topic-bloom-filters-for-read-optimization.md

### [REJECT] sstable-sparse-index-no-bloom
Duplicates existing belief `bloom-filter-not-integrated`
- Source: entries/2026/05/29/topic-bloom-filters-for-read-optimization.md

### [REJECT] bloom-filter-exists-but-unused
Duplicates existing belief `bloom-filter-not-integrated`
- Source: entries/2026/05/29/topic-bloom-filters-for-read-optimization.md

### [REJECT] size-tiered-compaction-overlapping-ranges
Duplicates existing belief `stcs-overlapping-key-ranges`
- Source: entries/2026/05/29/topic-bloom-filters-for-read-optimization.md

### [ACCEPT] digest-recomputed-on-preprepare
Backup replicas independently recompute the SHA-256 digest from the request payload in every PRE_PREPARE message; they never trust the primary's claimed digest value
- Source: entries/2026/05/29/topic-byzantine-fault-tolerance.md

### [ACCEPT] digest-binds-view-sequence-to-request
The `accepted_preprepare` dict enforces a one-to-one mapping from `(view, sequence)` to digest, preventing a Byzantine primary from assigning two different requests to the same protocol slot
- Source: entries/2026/05/29/topic-byzantine-fault-tolerance.md

### [ACCEPT] wrong-digest-cannot-reach-quorum
PREPARE and COMMIT messages with non-matching digests are silently dropped and never count toward quorum thresholds; a Byzantine node sending bad digests cannot contribute to agreement
- Source: entries/2026/05/29/topic-byzantine-fault-tolerance.md

### [ACCEPT] deterministic-serialization-enables-independent-verification
`json.dumps(sort_keys=True)` ensures all honest PBFT nodes produce identical SHA-256 digests for identical requests without inter-node coordination, transforming trust verification into a deterministic computation
- Source: entries/2026/05/29/topic-byzantine-fault-tolerance.md


### [ACCEPT] commit-only-in-batch
`OP_COMMIT` is written exclusively by `append_batch()` in `write-ahead-log/wal.py`; single `append()` calls never produce a COMMIT record, so individual operations are their own atomic unit
- Source: entries/2026/05/29/topic-checkpoint-vs-commit-semantics.md

### [REJECT] checkpoint-and-commit-force-sync
Covered by existing `wal-batch-always-fsyncs` and `force-true-bypasses-sync-mode`
- Source: entries/2026/05/29/topic-checkpoint-vs-commit-semantics.md

### [REJECT] replay-filters-markers
Covered by existing `wal-iterate-preserves-all-ops` and `wal-replay-ignores-commit`
- Source: entries/2026/05/29/topic-checkpoint-vs-commit-semantics.md

### [REJECT] checkpoint-seq-gates-truncate
Covered by existing `wal-checkpoint-returns-seq`, `wal-truncate-requires-explicit-seq`, and `checkpoint-record-unused-for-truncation`
- Source: entries/2026/05/29/topic-checkpoint-vs-commit-semantics.md

### [REJECT] batch-write-is-single-call
Duplicate of existing `wal-batch-single-write`
- Source: entries/2026/05/29/topic-checkpoint-vs-commit-semantics.md

### [REJECT] hash-index-deletes-before-rename
Duplicate of existing `delete-before-rename-ordering`
- Source: entries/2026/05/29/topic-compaction-input-deletion-ordering.md

### [REJECT] log-structured-no-fsync-on-write
Duplicate of existing `log-structured-bitcask-no-fsync`
- Source: entries/2026/05/29/topic-compaction-input-deletion-ordering.md

### [REJECT] hash-index-conditional-fsync
Duplicate of existing `bitcask-fsync-per-record-default`
- Source: entries/2026/05/29/topic-compaction-input-deletion-ordering.md

### [REJECT] neither-impl-fsyncs-directory
Duplicate of existing `no-directory-fsync-anywhere`
- Source: entries/2026/05/29/topic-compaction-input-deletion-ordering.md

### [ACCEPT] compact-closes-handles-before-keydir-fully-updated
In `hash-index-storage/bitcask.py`, cached file readers for old immutable files are closed at the start of the merge-write phase, before all keydir entries have been updated to point to new locations, creating a window where reads would fail even without concurrent access
- Source: entries/2026/05/29/topic-concurrent-merge-safety.md

### [ACCEPT] log-structured-compact-closes-active-file
In `log-structured-hash-table/bitcask.py`, compaction closes `_active_file` during its write phase, making all reads and writes to the active segment impossible until the active file is reopened under a new name at the end of compaction
- Source: entries/2026/05/29/topic-concurrent-merge-safety.md

### [REJECT] neither-bitcask-has-atomic-switchover
Covered by existing `delete-before-rename-ordering`, `bitcask-compact-not-crash-safe`, and `compaction-not-atomic`
- Source: entries/2026/05/29/topic-concurrent-merge-safety.md

### [REJECT] crash-between-delete-and-rename-loses-data
Duplicate of existing `bitcask-compact-not-crash-safe` and `bitcask-compaction-not-crash-safe`
- Source: entries/2026/05/29/topic-concurrent-merge-safety.md

### [REJECT] bitcask-crc-excludes-header
Duplicate of existing `bitcask-crc-covers-payload-only`
- Source: entries/2026/05/29/topic-crc-coverage-audit.md

### [REJECT] wal-crc-excludes-seq-and-lengths
Covered by existing `wal-crc-does-not-cover-seqnum` and `wal-protects-data-not-metadata`
- Source: entries/2026/05/29/topic-crc-coverage-audit.md

### [ACCEPT] btree-wal-checksum-excludes-page-num
In `b-tree-storage-engine/btree.py`, the WAL entry checksum covers only `page_data`, so a corrupted `page_num` during recovery writes a valid page to the wrong disk location without detection — the most dangerous integrity gap in the codebase
- Source: entries/2026/05/29/topic-crc-coverage-audit.md

### [ACCEPT] lsm-and-sstable-have-no-checksums
Neither `log-structured-merge-tree/lsm.py` nor `sstable-and-compaction/sstable.py` compute or verify any checksums; a single bit-flip in a length-prefix field causes cascading misframing of all subsequent records
- Source: entries/2026/05/29/topic-crc-coverage-audit.md

### [REJECT] no-component-has-full-record-crc
Subsumed by existing `all-three-implementations-use-payload-only-crc` combined with the new `lsm-and-sstable-have-no-checksums`
- Source: entries/2026/05/29/topic-crc-coverage-audit.md

### [ACCEPT] crc32-covers-records-not-files
CRC32 is computed per individual record or page (typically hundreds of bytes to a few KB), not per file; file-level corruption is detected only indirectly by failing to decode a record, not by any file-wide checksum
- Source: entries/2026/05/29/topic-crc32-vs-xxhash-choice.md

### [REJECT] sstable-lacks-checksums
Covered by accepted `lsm-and-sstable-have-no-checksums` which includes both components
- Source: entries/2026/05/29/topic-crc32-vs-xxhash-choice.md

### [REJECT] crc32-input-excludes-metadata
Covered by existing `wal-crc-does-not-cover-seqnum`, `wal-protects-data-not-metadata`, and `bitcask-crc-covers-payload-only`
- Source: entries/2026/05/29/topic-crc32-vs-xxhash-choice.md

### [ACCEPT] hashlib-reserved-for-crypto-distribution
The codebase consistently uses `zlib.crc32` for corruption detection in storage engines and `hashlib` (SHA-256) for content addressing (Merkle trees), uniform distribution (consistent hashing, bloom filters), and cryptographic integrity (BFT) — the two hash families never cross purposes
- Source: entries/2026/05/29/topic-crc32-vs-xxhash-choice.md


### [ACCEPT] hash-index-all-keys-in-memory
Both Bitcask implementations keep every live key in an in-memory dict (`keydir` / `_index`), meaning the key set must fit in RAM; there is no disk-based fallback for partial index spill.
- Source: entries/2026/05/29/topic-ddia-chapter3-hash-indexes.md

### [REJECT] sstable-sparse-index-reduces-memory
Already captured by `sstable-sparse-index-every-nth` and `lsm-sparse-index-default-16`.
- Source: entries/2026/05/29/topic-ddia-chapter3-hash-indexes.md

### [ACCEPT] hash-index-no-range-queries
Neither hash index implementation (`hash-index-storage/bitcask.py` or `log-structured-hash-table/bitcask.py`) provides a `scan`, `range`, or ordered-iteration method; only exact-key point lookups are supported.
- Source: entries/2026/05/29/topic-ddia-chapter3-hash-indexes.md

### [ACCEPT] lsm-requires-wal-hash-does-not
The LSM tree uses a separate WAL (`lsm.py:14-64`) for crash recovery because its memtable is volatile; the hash index writes directly to the append-only data file, which serves as its own recovery log and needs no WAL.
- Source: entries/2026/05/29/topic-ddia-chapter3-hash-indexes.md

### [REJECT] crc-integrity-only-in-log-structured-variant
Already captured by `bitcask-no-checksum-validation` (hash-index-storage has no CRC) and `bitcask-crc-covers-payload-only` (log-structured variant does).
- Source: entries/2026/05/29/topic-ddia-chapter3-hash-indexes.md

### [REJECT] no-dir-fsync-anywhere
Already exists as `no-directory-fsync-anywhere`.
- Source: entries/2026/05/29/topic-directory-fsync-after-rename.md

### [REJECT] rename-without-durability
Already exists as `rename-without-dir-barrier`.
- Source: entries/2026/05/29/topic-directory-fsync-after-rename.md

### [REJECT] file-data-fsync-is-correct
Already captured by `all-fsync-sites-data-integrity-only`.
- Source: entries/2026/05/29/topic-directory-fsync-after-rename.md

### [REJECT] wal-rotation-creates-unfsynced-entries
Already exists as `wal-no-directory-fsync`.
- Source: entries/2026/05/29/topic-directory-fsync-after-rename.md

### [REJECT] allocate-page-not-wal-protected
Already exists as `btree-allocate-page-outside-wal`.
- Source: entries/2026/05/29/topic-free-list-corruption-risks.md

### [ACCEPT] write-meta-no-fsync
`PageManager._write_meta` calls `flush()` but not `os.fsync()`, so metadata updates (`next_free_page`, `free_list_head`) are not durable against power loss — compounding the WAL bypass with a durability gap on the direct write path.
- Source: entries/2026/05/29/topic-free-list-corruption-risks.md

### [REJECT] wal-recover-ignores-metadata
Already exists as `wal-protects-data-not-metadata`.
- Source: entries/2026/05/29/topic-free-list-corruption-risks.md

### [ACCEPT] no-leak-detection-mechanism
The B-tree has no allocation bitmap, page-reachability check, or post-recovery consistency scan to detect leaked or orphaned pages; once a page is lost between the tree and the free list, it is silently unrecoverable.
- Source: entries/2026/05/29/topic-free-list-corruption-risks.md

### [REJECT] free-page-non-atomic
Covered by existing `btree-free-page-bypasses-wal` (root cause) and `btree-delete-leaks-pages` (consequence).
- Source: entries/2026/05/29/topic-free-list-corruption-risks.md

### [REJECT] free-list-is-intrusive-singly-linked
Already exists as `btree-free-list-intrusive`.
- Source: entries/2026/05/29/topic-free-list-reuse-under-churn.md

### [REJECT] allocate-prefers-free-list
Already exists as `allocate-page-prefers-free-list`.
- Source: entries/2026/05/29/topic-free-list-reuse-under-churn.md

### [ACCEPT] next-free-page-is-monotonic
The `next_free_page` metadata field only ever increases — it is the high-water mark of file extension, never decremented even when pages are freed; file size never shrinks, only interior pages are recycled via the free list.
- Source: entries/2026/05/29/topic-free-list-reuse-under-churn.md

### [ACCEPT] free-page-overwrites-content
`PageManager.free_page()` overwrites the freed page's content with a zero header and a free-list pointer, destroying the original leaf/internal node data so freed pages cannot be accidentally read as valid tree nodes.
- Source: entries/2026/05/29/topic-free-list-reuse-under-churn.md

### [REJECT] two-rename-sites-in-compaction
Covered by `rename-without-dir-barrier`; the specific count/location of rename call sites is unstable detail.
- Source: entries/2026/05/29/topic-fsync-directory-durability.md

### [REJECT] wal-rotation-creates-files-without-dir-fsync (entry 5)
Already exists as `wal-no-directory-fsync`.
- Source: entries/2026/05/29/topic-fsync-directory-durability.md

### [REJECT] file-content-fsync-is-thorough
Already captured by `all-fsync-sites-data-integrity-only`.
- Source: entries/2026/05/29/topic-fsync-directory-durability.md


### [REJECT] wal-flush-fsync-pairing
Covered by existing `rotation-always-fsyncs`, `wal-batch-always-fsyncs`, and `wal-commit-sync-before-truncate` which together establish the consistent pairing across all WAL sync points.
- Source: entries/2026/05/29/topic-fsync-durability-guarantees.md

### [REJECT] lsm-wal-no-fsync
Duplicate of existing `lsm-wal-has-no-fsync`.
- Source: entries/2026/05/29/topic-fsync-durability-guarantees.md

### [REJECT] no-directory-fsync
Duplicate of existing `no-directory-fsync-anywhere`.
- Source: entries/2026/05/29/topic-fsync-durability-guarantees.md

### [REJECT] no-platform-specific-sync
The F_FULLFSYNC claim is covered by existing `apfs-masks-linux-fsync-bugs`; the fdatasync claim is captured below as `fdatasync-never-used`.
- Source: entries/2026/05/29/topic-fsync-durability-guarantees.md

### [REJECT] batch-sync-commit-override
Covered by existing `force-true-bypasses-sync-mode` and `wal-batch-always-fsyncs`.
- Source: entries/2026/05/29/topic-fsync-durability-guarantees.md

### [REJECT] lsm-wal-missing-fsync
Duplicate of existing `lsm-wal-has-no-fsync`.
- Source: entries/2026/05/29/topic-fsync-flush-distinction.md

### [REJECT] log-hash-table-missing-fsync
Duplicate of existing `log-structured-bitcask-no-fsync`.
- Source: entries/2026/05/29/topic-fsync-flush-distinction.md

### [REJECT] flush-only-no-durability
General OS/systems principle, not a codebase-specific invariant. Well-known prerequisite knowledge, not a claim about this code that could break if violated.
- Source: entries/2026/05/29/topic-fsync-flush-distinction.md

### [REJECT] correct-sync-pattern
Already captured by individual per-implementation beliefs: `rotation-always-fsyncs`, `wal-batch-always-fsyncs`, `btree-wal-before-data`, `bitcask-fsync-per-record-default`.
- Source: entries/2026/05/29/topic-fsync-flush-distinction.md

### [ACCEPT] fdatasync-never-used
No implementation uses `os.fdatasync()`, missing a safe optimization for append-only WALs where only data (not metadata like mtime) needs to reach disk.
- Source: entries/2026/05/29/topic-fsync-flush-distinction.md

### [REJECT] lsm-wal-never-fsyncs
Duplicate of existing `lsm-wal-has-no-fsync`.
- Source: entries/2026/05/29/topic-fsync-guarantees-across-implementations.md

### [REJECT] btree-durability-via-wal-commit
Covered by existing `btree-wal-no-data-fsync` (page writes intentionally not fsynced) and `wal-commit-sync-before-truncate` (commit is the durability barrier).
- Source: entries/2026/05/29/topic-fsync-guarantees-across-implementations.md

### [REJECT] wal-py-always-pairs-flush-fsync
Already captured distributively by `rotation-always-fsyncs`, `wal-batch-always-fsyncs`, `wal-truncate-fsyncs-before-scan`, and `wal-commit-sync-before-truncate`.
- Source: entries/2026/05/29/topic-fsync-guarantees-across-implementations.md

### [REJECT] bitcask-sync-writes-flag-controls-fsync
Duplicate of existing `bitcask-fsync-per-record-default`.
- Source: entries/2026/05/29/topic-fsync-guarantees-across-implementations.md

### [REJECT] log-structured-hash-table-no-fsync
Duplicate of existing `log-structured-bitcask-no-fsync`.
- Source: entries/2026/05/29/topic-fsync-guarantees-across-implementations.md

### [ACCEPT] sstable-writer-finish-no-fsync
`sstable-and-compaction/sstable.py:SSTableWriter.finish()` closes the file without calling `flush()` or `os.fsync()`, relying on implicit Python close-time buffer flush with no disk durability guarantee.
- Source: entries/2026/05/29/topic-fsync-guarantees-across-implementations.md

### [REJECT] no-dir-fsync-anywhere
Duplicate of existing `no-directory-fsync-anywhere`.
- Source: entries/2026/05/29/topic-fsync-on-directory.md

### [REJECT] hint-file-loss-is-safe
Covered by existing `hint-files-are-optimization-only` which establishes that hints carry no unique data.
- Source: entries/2026/05/29/topic-fsync-on-directory.md

### [REJECT] data-file-dir-entry-same-gap-as-hint
Derivable from existing `no-directory-fsync-anywhere` (the gap is universal) combined with `hint-files-are-optimization-only` (hints are safe to lose, data files are not).
- Source: entries/2026/05/29/topic-fsync-on-directory.md

### [REJECT] lsm-flush-double-gap
Derivable from existing `lsm-wal-truncated-after-sstable-write` (ordering) + `no-directory-fsync-anywhere` (neither directory entry durable) + `lsm-wal-truncate-destroys-immediately` (WAL gone immediately).
- Source: entries/2026/05/29/topic-fsync-on-directory.md

### [REJECT] no-temp-rename-pattern
Duplicate of existing `no-atomic-file-creation`.
- Source: entries/2026/05/29/topic-fsync-on-directory.md

### [REJECT] wal-rotate-no-dir-fsync
Duplicate of existing `wal-no-directory-fsync`.
- Source: entries/2026/05/29/topic-fsync-on-new-file-creation.md

### [ACCEPT] wal-recovery-depends-on-listdir
WAL segment discovery during recovery uses `os.listdir()`, so any segment whose parent directory entry was not fsynced becomes invisible after a crash — connecting the dir-fsync gap to concrete data loss.
- Source: entries/2026/05/29/topic-fsync-on-new-file-creation.md

### [REJECT] rotate-called-from-three-paths
Call-graph detail that establishes blast radius but is derivable from `wal-no-directory-fsync` (which already implies all rotation-dependent write paths are affected) and is fragile to refactoring.
- Source: entries/2026/05/29/topic-fsync-on-new-file-creation.md


### [REJECT] wal-replay-stops-at-corruption
Already covered by existing `wal-crc-mismatch-halts-all-replay`, `wal-no-resync-after-corruption`, and `wal-corruption-stops-per-file`
- Source: entries/2026/05/29/topic-gap-detection-during-replay.md

### [REJECT] no-sequence-continuity-check
Already covered by existing `wal-no-gap-detection` and `recover-seq-no-gap-detection`
- Source: entries/2026/05/29/topic-gap-detection-during-replay.md

### [ACCEPT] lsm-replay-strips-seq-nums
The LSM tree's `replay` method (`lsm.py:28`) returns `List[Tuple[str, bytes]]`, discarding WAL sequence numbers and making gap detection impossible at the LSM layer
- Source: entries/2026/05/29/topic-gap-detection-during-replay.md

### [REJECT] recover-seq-num-vulnerable-to-early-corruption
Already covered by existing `recover-seq-skips-corrupt-files`
- Source: entries/2026/05/29/topic-gap-detection-during-replay.md

### [ACCEPT] hash-index-no-crc-by-design
`hash-index-storage/bitcask.py` intentionally omits CRC to focus on keydir/append-log/compaction; `log-structured-hash-table/bitcask.py` provides the integrity-checking variant with `zlib.crc32` and `CorruptionError`
- Source: entries/2026/05/29/topic-hash-index-bitcask-no-crc.md

### [ACCEPT] silent-corruption-returns-garbage
A bit-flip in a value region of `hash-index-storage/bitcask.py` causes `get()` to return corrupted data with no error; the only partial integrity check is the key-equality assertion
- Source: entries/2026/05/29/topic-hash-index-bitcask-no-crc.md

### [REJECT] crc-mismatch-halts-scan
Already covered by existing `corruption-terminates-scan` and `bitcask-crc32-raises-corruption-error`
- Source: entries/2026/05/29/topic-hash-index-bitcask-no-crc.md

### [ACCEPT] compaction-propagates-corruption
`hash-index-storage/bitcask.py:compact` copies records without integrity validation, so silently corrupted data survives compaction into new files
- Source: entries/2026/05/29/topic-hash-index-bitcask-no-crc.md

### [ACCEPT] header-corruption-derails-sequential-scan
A corrupted `key_size` or `val_size` in `hash-index-storage/bitcask.py` causes `_scan_data_file` to read wrong byte boundaries, potentially misinterpreting all subsequent records in that file
- Source: entries/2026/05/29/topic-hash-index-bitcask-no-crc.md

### [REJECT] immutable-memtables-never-populated
Already covered by existing `flush-skips-immutable-memtable-stage`
- Source: entries/2026/05/29/topic-immutable-memtable-gap.md

### [REJECT] flush-bypasses-freeze-step
Already covered by existing `flush-skips-immutable-memtable-stage`
- Source: entries/2026/05/29/topic-immutable-memtable-gap.md

### [ACCEPT] read-path-anticipates-concurrent-flush
Both `get()` (line 254) and `range_scan()` (line 286) check `_immutable_memtables` in the correct order, indicating the read path was designed for a concurrent-flush architecture that the write path never implements
- Source: entries/2026/05/29/topic-immutable-memtable-gap.md

### [ACCEPT] single-threaded-masks-immutable-gap
In the synchronous single-threaded LSM execution model, the missing immutable memtable lifecycle step causes no observable data loss because `_flush()` completes atomically within the caller's execution flow
- Source: entries/2026/05/29/topic-immutable-memtable-gap.md

### [REJECT] no-concurrency-control
Already covered by existing `no-concurrency-primitives`
- Source: entries/2026/05/29/topic-latch-coupling-postgres.md

### [REJECT] sibling-links-for-scans-not-splits
Already covered by existing `sibling-field-is-structural-only` and `range-scan-follows-sibling-chain`
- Source: entries/2026/05/29/topic-latch-coupling-postgres.md

### [ACCEPT] btree-fix-plan-all-crash-safety
All six bugs documented in `fix-plan.md` concern single-writer WAL/fsync crash safety; none involve concurrent access races, illustrating that even without concurrency the WAL protocol has subtle correctness requirements
- Source: entries/2026/05/29/topic-latch-coupling-postgres.md

### [REJECT] wal-provides-durability-not-isolation
Already covered by existing `wal-provides-crash-safety-not-concurrency`
- Source: entries/2026/05/29/topic-latch-coupling-postgres.md

### [REJECT] dynamo-no-delete-method
Already covered by existing `dynamo-no-delete-support`
- Source: entries/2026/05/29/topic-leaderless-deletion-gap.md

### [ACCEPT] read-repair-resurrects-deleted-keys
`DynamoCluster.get()` read repair pushes the highest-versioned value to stale replicas, so a naive delete (removing from `_store`) is undone when a recovering node still holds the old value
- Source: entries/2026/05/29/topic-leaderless-deletion-gap.md

### [ACCEPT] anti-entropy-unions-all-keys
`anti_entropy_repair()` collects keys from all available nodes via set union, so a key present on any single node is propagated to all other nodes regardless of whether it was intentionally removed elsewhere
- Source: entries/2026/05/29/topic-leaderless-deletion-gap.md

### [ACCEPT] versioned-value-no-tombstone-flag
The `VersionedValue` dataclass (`dynamo.py:14-18`) carries only `value`, `version`, and `node_id` with no field to distinguish a live value from a deletion marker, making correct distributed deletes impossible without schema changes
- Source: entries/2026/05/29/topic-leaderless-deletion-gap.md


### [ACCEPT] leaf-split-updates-sibling-pointers
When a leaf splits, the new page inherits the old page's `next_sibling` pointer and the old page's `next_sibling` is updated to point to the new page — two field writes during an operation that already rewrites both pages
- Source: entries/2026/05/29/topic-leaf-sibling-chains.md

### [REJECT] btree-leaf-pages-have-forward-sibling-pointer
Duplicates existing beliefs `btree-leaf-sibling-chain` and `leaf-wire-format-is-header-sibling-entries`
- Source: entries/2026/05/29/topic-leaf-sibling-chains.md

### [REJECT] range-scan-walks-sibling-chain
Duplicates existing belief `range-scan-follows-sibling-chain`
- Source: entries/2026/05/29/topic-leaf-sibling-chains.md

### [REJECT] wal-protects-sibling-pointer-consistency
Duplicates existing belief `btree-wal-provides-split-atomicity`
- Source: entries/2026/05/29/topic-leaf-sibling-chains.md

### [REJECT] delete-before-rename-in-bitcask
Duplicates existing belief `delete-before-rename-ordering`
- Source: entries/2026/05/29/topic-leveldb-manifest-pattern.md

### [ACCEPT] lsm-forward-only-iteration
SSTable iterators (`scan`, `scan_all`) support only forward iteration; there is no `Prev()` or reverse scan capability, preventing `ORDER BY DESC` or backward cursor pagination
- Source: entries/2026/05/29/topic-leveldb-merging-iterator.md

### [ACCEPT] sstable-has-timestamps-lsm-ignores-them
`SSTableEntry` in `sstable.py` carries a `timestamp` field, but `lsm.py`'s merge and conflict resolution uses only structural SSTable ordering (sequence number priority), not per-entry timestamps
- Source: entries/2026/05/29/topic-leveldb-merging-iterator.md

### [ACCEPT] lsm-two-merge-strategies
Compaction uses `heapq`-based k-way merge with `prev_key` deduplication (lsm.py ~line 323), while `range_scan` uses a dict-based materialize-everything approach (lines 275–298) — two fundamentally different merge strategies in the same codebase
- Source: entries/2026/05/29/topic-leveldb-merging-iterator.md

### [REJECT] lsm-no-snapshot-isolation
Covered by existing beliefs `iterate-not-snapshot-isolated` and `range-scan-no-concurrency-protection`
- Source: entries/2026/05/29/topic-leveldb-merging-iterator.md

### [ACCEPT] lsm-tree-has-no-level-concept
`LSMTree` in lsm.py maintains a flat list of SSTables ordered by sequence number with no level assignment; compaction merges all files into one rather than promoting between levels
- Source: entries/2026/05/29/topic-leveldb-version-set.md

### [ACCEPT] fresh-sstables-always-start-at-level-zero
`SSTableWriter.finish()` hardcodes `level=0` (sstable.py:104), matching LevelDB's invariant that flushed memtables always produce L0 files
- Source: entries/2026/05/29/topic-leveldb-version-set.md

### [REJECT] sstable-metadata-tracks-key-range-and-level
Field existence is already implied by `sstable-metadata-level-field-unused`; the key range tracking is a minor structural detail
- Source: entries/2026/05/29/topic-leveldb-version-set.md

### [REJECT] no-manifest-or-durable-version-tracking
Duplicates existing belief `lsm-no-manifest`
- Source: entries/2026/05/29/topic-leveldb-version-set.md

### [REJECT] leveled-compaction-promotes-to-next-level
Duplicates existing belief `leveled-compaction-promotes-to-l1`
- Source: entries/2026/05/29/topic-leveldb-version-set.md

### [ACCEPT] live-projection-uses-subscriber-list
`LiveProjection` receives automatic updates by registering a callback in `EventStore._subscribers` (a list of `Callable[[Event], None]]`), which is invoked synchronously inside `append()` and `append_batch()`
- Source: entries/2026/05/29/topic-live-projection-subscription-mechanism.md

### [ACCEPT] subscriber-dispatch-is-synchronous
Event subscriber notification in `EventStore` blocks the `append()` caller — projections update on the writer's thread before the append call returns, trading throughput for zero-gap consistency
- Source: entries/2026/05/29/topic-live-projection-subscription-mechanism.md

### [ACCEPT] batch-notifies-after-all-stored
`append_batch()` stores all events in the batch first, then notifies subscribers of each event sequentially, ensuring subscribers see a consistent state where all batch events exist
- Source: entries/2026/05/29/topic-live-projection-subscription-mechanism.md

### [ACCEPT] catch-up-then-subscribe
`LiveProjection` implements the catch-up subscription pattern: replay historical events via `read_all(from_position=...)`, then switch to push-based updates for future events, bridging pull/push at a known position with no gap
- Source: entries/2026/05/29/topic-live-projection-subscription-mechanism.md


### [ACCEPT] wal-truncate-rewrites-files
`WriteAheadLog.truncate()` reads and rewrites every segment file record-by-record rather than deleting whole segment files, making it O(total records) instead of O(segments)
- Source: entries/2026/05/29/topic-log-compaction-vs-persistence.md

### [ACCEPT] checkpoint-record-not-used-in-recovery
`OP_CHECKPOINT` records are written to the WAL but recovery (`_recover_seq_num`) does not search for them; the caller must supply the checkpoint sequence number externally via `replay(after_seq=)`
- Source: entries/2026/05/29/topic-log-compaction-vs-persistence.md

### [REJECT] lsm-wal-truncate-is-full-wipe
Duplicated by existing `lsm-wal-truncate-destroys-immediately` and `lsm-wal-truncate-no-arg` which together capture that the LSM WAL truncation unconditionally wipes the file with no arguments
- Source: entries/2026/05/29/topic-log-compaction-vs-persistence.md

### [REJECT] no-fsync-before-wal-truncation
Already covered by existing `btree-wal-truncate-before-data-sync` and `lsm-compaction-deletes-without-fsync-barrier` which document this missing fsync gap in both engines
- Source: entries/2026/05/29/topic-log-compaction-vs-persistence.md

### [ACCEPT] cbf-saturated-counters-are-permanent
When a `CountingBloomFilter` counter reaches `_max_val` (default 15), it is never decremented, preventing false negatives at the cost of a slight false positive rate increase
- Source: entries/2026/05/29/topic-memtable-bloom-filter-lifecycle.md

### [ACCEPT] cbf-remove-raises-on-absent-item
`CountingBloomFilter.remove()` raises `ValueError` if any hash position has a zero counter, providing a safety check against removing items that were never added
- Source: entries/2026/05/29/topic-memtable-bloom-filter-lifecycle.md

### [REJECT] lsm-memtable-has-no-bloom-filter
Duplicated by existing `bloom-filter-not-integrated` which already captures that the bloom filter is not wired into the LSM tree
- Source: entries/2026/05/29/topic-memtable-bloom-filter-lifecycle.md

### [ACCEPT] cbf-8x-memory-vs-standard
`CountingBloomFilter` uses one byte per counter position (`bytearray(self._m)`) versus one bit per position in `BloomFilter`, an 8x memory overhead for deletion support
- Source: entries/2026/05/29/topic-memtable-bloom-filter-lifecycle.md

### [ACCEPT] mvcc-append-only-versions
Writers never modify existing `Version` objects (except self-overwrites); new values are appended to per-key version lists, preserving the immutable snapshot for concurrent readers
- Source: entries/2026/05/29/topic-mvcc-snapshot-isolation.md

### [ACCEPT] visibility-requires-three-conditions
A version is visible to a transaction only if its creator committed, was not in the reader's `active_at_start` set, and has a lower `tx_id` — all three must hold
- Source: entries/2026/05/29/topic-mvcc-snapshot-isolation.md

### [ACCEPT] first-committer-wins-on-write-conflict
When two concurrent transactions write the same key, the first to call `commit()` succeeds and the second is aborted automatically
- Source: entries/2026/05/29/topic-mvcc-snapshot-isolation.md

### [ACCEPT] gc-preserves-active-snapshots
Garbage collection removes only versions unreachable by any active transaction, ensuring long-running read-only transactions see consistent data
- Source: entries/2026/05/29/topic-mvcc-snapshot-isolation.md

### [ACCEPT] ssi-extends-mvcc-with-dependency-tracking
`SSIDatabase` adds a `_dependency_graph` and `_predicate_locks` on top of MVCC to detect read-write conflicts (write skew) that basic snapshot isolation permits
- Source: entries/2026/05/29/topic-mvcc-snapshot-isolation.md

### [REJECT] lsm-compact-no-snapshot-safety
Duplicated by existing `compact-no-concurrency-safety` which already captures that compaction has no protection for concurrent operations
- Source: entries/2026/05/29/topic-mvcc-version-lifetime.md

### [REJECT] mvcc-visibility-is-logical-superversion
This is an analogy between two systems rather than a testable factual claim about the code — it asserts conceptual equivalence rather than observable behavior
- Source: entries/2026/05/29/topic-mvcc-version-lifetime.md

### [ACCEPT] no-refcount-in-codebase
No file in the repository implements reference counting on SSTable files, Version objects, or any storage-layer snapshot structure
- Source: entries/2026/05/29/topic-mvcc-version-lifetime.md

### [REJECT] lsm-scan-race-window
Duplicated by existing `range-scan-no-concurrency-protection` which already documents the lack of concurrency safety for range scans
- Source: entries/2026/05/29/topic-mvcc-version-lifetime.md

### [ACCEPT] no-o-direct-usage
No file in the ddia-implementations codebase uses `O_DIRECT`, `os.open()`, or any `os.O_` flags; all I/O goes through Python's buffered `open()` and the kernel page cache
- Source: entries/2026/05/29/topic-o-direct-and-dio.md

### [REJECT] fsync-is-durability-primitive
Too broad — the role of fsync is already documented by multiple specific existing beliefs including `all-fsync-sites-data-integrity-only`, `fsync-used-only-for-appends`, `batch-mode-defers-fsync`, and per-engine fsync beliefs
- Source: entries/2026/05/29/topic-o-direct-and-dio.md

### [REJECT] wal-three-sync-modes
Already covered by existing `batch-mode-defers-fsync` and `none-mode-never-syncs-on-append` which document the individual sync mode behaviors
- Source: entries/2026/05/29/topic-o-direct-and-dio.md

### [ACCEPT] btree-page-writes-unbuffered-by-app
`PageManager` has no application-level page cache — every `read_page` reads from the file and every `write_page` writes immediately, relying entirely on the kernel page cache for performance
- Source: entries/2026/05/29/topic-o-direct-and-dio.md


### [ACCEPT] serialization-has-no-overflow-check
`_serialize_leaf` packs all entries into a buffer without comparing its length to `page_size`; overflow prevention is entirely the caller's responsibility
- Source: entries/2026/05/29/topic-page-overflow-and-size-limits.md

### [ACCEPT] write-page-silently-truncates
`PageManager.write_page` truncates data exceeding `page_size` to exactly `page_size` bytes without raising an error, which would corrupt the node header's `num_keys` count
- Source: entries/2026/05/29/topic-page-overflow-and-size-limits.md

### [ACCEPT] max-keys-prevents-overflow
The `max_keys_per_page` parameter bounds entries per node so that serialized output fits within `page_size`, with splits triggered before the limit is exceeded
- Source: entries/2026/05/29/topic-page-overflow-and-size-limits.md

### [ACCEPT] single-kv-size-validated
`BTree.put` raises `ValueError` for any single key-value pair that cannot fit in a page, blocking the case where even one entry would overflow
- Source: entries/2026/05/29/topic-page-overflow-and-size-limits.md

### [ACCEPT] btree-free-page-no-parent-fixup
`PageManager.free_page` adds a page to the free list without removing the parent's downlink to it, creating a dangling-pointer risk on crash
- Source: entries/2026/05/29/topic-postgres-nbtree-half-dead-state-machine.md

### [REJECT] btree-no-merge-or-rebalance
Duplicate of existing `btree-no-underflow-rebalancing`
- Source: entries/2026/05/29/topic-postgres-nbtree-half-dead-state-machine.md

### [ACCEPT] btree-wal-no-operation-boundaries
The WAL logs individual page writes and commits by full truncation; it cannot represent a multi-step structural operation as an atomic unit
- Source: entries/2026/05/29/topic-postgres-nbtree-half-dead-state-machine.md

### [ACCEPT] btree-single-threaded-assumption
The B-tree implementation has no locking, latching, or concurrency control; all crash-safety gaps exist even without concurrent access
- Source: entries/2026/05/29/topic-postgres-nbtree-half-dead-state-machine.md

### [ACCEPT] no-pytest-config-exists
The repository contains no `pyproject.toml`, `pytest.ini`, `setup.cfg`, or root `conftest.py`; pytest runs with pure default settings
- Source: entries/2026/05/29/topic-pytest-configuration.md

### [ACCEPT] tester-test-files-not-excluded
No `collect_ignore`, `norecursedirs`, `testpaths`, or `python_files` directive exists anywhere in the repo, so `tester_test_*.py` files will be collected by pytest's default `*_test.py` glob
- Source: entries/2026/05/29/topic-pytest-configuration.md

### [REJECT] gitignore-is-minimal
Trivial project scaffolding detail; not an architectural invariant a developer needs to know
- Source: entries/2026/05/29/topic-pytest-configuration.md

### [REJECT] default-pytest-collection-applies
Redundant with `no-pytest-config-exists` — the default collection behavior is a direct consequence of the absence of configuration
- Source: entries/2026/05/29/topic-pytest-configuration.md

### [REJECT] wal-stop-at-first-error
Duplicate of existing `wal-crc-mismatch-halts-all-replay` and `corruption-terminates-scan`
- Source: entries/2026/05/29/topic-record-boundary-recovery.md

### [REJECT] wal-no-resync-capability
Duplicate of existing `wal-no-resync-after-corruption`
- Source: entries/2026/05/29/topic-record-boundary-recovery.md

### [ACCEPT] lsm-wal-no-crc
The LSM tree's WAL (`lsm.py:14-53`) uses only length-prefixed framing with no checksum, making it unable to distinguish corruption from valid data unless a length field causes an out-of-bounds read
- Source: entries/2026/05/29/topic-record-boundary-recovery.md

### [ACCEPT] segment-rotation-limits-corruption-blast-radius
Both Bitcask implementations rotate to new segment files at a configurable size threshold, bounding the maximum data loss from stop-at-first-error to the records trailing the corruption within a single segment
- Source: entries/2026/05/29/topic-record-boundary-recovery.md

### [REJECT] bitcask-compaction-rename-not-durable
Duplicate of existing `rename-without-dir-barrier`
- Source: entries/2026/05/29/topic-rename-durability-guarantees.md

### [REJECT] wal-rotation-same-gap
Duplicate of existing `wal-no-directory-fsync`
- Source: entries/2026/05/29/topic-rename-durability-guarantees.md

### [REJECT] file-content-fsync-is-thorough
Duplicate of existing `all-fsync-sites-data-integrity-only`
- Source: entries/2026/05/29/topic-rename-durability-guarantees.md


### [ACCEPT] pending-requeue-enables-multihop
Every accepted remote change in `apply_remote_change` is appended to `_pending`, causing it to be forwarded to the next node in the replication topology during the next sync cycle
- Source: entries/2026/05/29/topic-ring-topology-propagation.md

### [ACCEPT] seen-set-keyed-by-ts-origin
`_seen` maps each key to a set of `(timestamp, origin_node_id)` tuples; a change is skipped if its exact `(ts, origin)` pair is already in the set for that key
- Source: entries/2026/05/29/topic-ring-topology-propagation.md

### [ACCEPT] local-writes-self-immunize
Both `put` and `delete` call `_record_seen` with the local node's ID, ensuring the originating node drops its own changes when they loop back through the ring
- Source: entries/2026/05/29/topic-ring-topology-propagation.md

### [REJECT] custom-merge-creates-new-identity
Duplicates existing belief `multi-leader-custom-merge-new-timestamp`
- Source: entries/2026/05/29/topic-ring-topology-propagation.md

### [ACCEPT] get-pending-drains-atomically
`get_pending_changes` returns the current `_pending` list and replaces it with an empty list, ensuring each change is delivered exactly once per replication cycle
- Source: entries/2026/05/29/topic-ring-topology-propagation.md

### [ACCEPT] lsm-flat-compaction-always-bottommost
`LSMTree.compact()` in `lsm.py` merges all SSTables into one without level hierarchy, making every compaction implicitly a bottommost-level operation where tombstone removal is always safe
- Source: entries/2026/05/29/topic-rocksdb-bottommost-compaction.md

### [REJECT] sstable-merge-tombstone-flag-is-caller-controlled
Duplicates existing belief `merge-sstables-tombstone-flag-is-caller-controlled`
- Source: entries/2026/05/29/topic-rocksdb-bottommost-compaction.md

### [REJECT] leveled-compaction-promotes-to-level-1
Duplicates existing belief `leveled-compaction-promotes-to-l1`
- Source: entries/2026/05/29/topic-rocksdb-bottommost-compaction.md

### [REJECT] no-bottommost-level-detection-exists
Duplicates existing belief `sstable-compaction-manager-lacks-bottommost-detection`
- Source: entries/2026/05/29/topic-rocksdb-bottommost-compaction.md

### [REJECT] seq-num-corruption-causes-silent-data-loss
Duplicates existing belief `seq-num-corruption-is-silent` — the specific truncate/replay failure modes are consequences of the already-documented gap
- Source: entries/2026/05/29/topic-seq-num-integrity-gap.md

### [ACCEPT] seq-num-crc-fix-is-zero-cost
Adding `seq_num` to the CRC input requires changing two lines (`_encode_record` and `_read_record`) with no measurable runtime overhead, since CRC32 over 9 extra bytes is negligible
- Source: entries/2026/05/29/topic-seq-num-integrity-gap.md

### [REJECT] record-length-corruption-is-implicitly-caught
Duplicates existing belief `record-length-implicitly-validated`
- Source: entries/2026/05/29/topic-seq-num-integrity-gap.md

### [ACCEPT] wal-format-change-breaks-compatibility
Changing the CRC input in `_encode_record` invalidates all existing WAL files; a production deployment would require a version byte and dual-path CRC verification during migration
- Source: entries/2026/05/29/topic-seq-num-integrity-gap.md

### [ACCEPT] snapshots-are-in-memory-only
Projection snapshots are stored in a dict on the `EventStore` instance and do not persist to disk; they are lost on process restart
- Source: entries/2026/05/29/topic-snapshot-storage-format.md

### [ACCEPT] snapshots-dict-is-monkey-patched
The `_snapshots` attribute is not declared on `EventStore`; it is lazily created by `Projection.save_snapshot` via `hasattr`/setattr on the store instance
- Source: entries/2026/05/29/topic-snapshot-storage-format.md

### [ACCEPT] snapshot-keyed-by-projection-name
Each projection's snapshot is stored under `self.name` in `_store._snapshots`, so two projections with the same name on the same store will overwrite each other
- Source: entries/2026/05/29/topic-snapshot-storage-format.md

### [ACCEPT] save-snapshot-deep-copies-state
`save_snapshot` uses `copy.deepcopy` to isolate the stored snapshot from subsequent mutations to `_state`
- Source: entries/2026/05/29/topic-snapshot-storage-format.md

### [REJECT] event-log-persists-but-snapshots-do-not
Redundant given `snapshots-are-in-memory-only` (proposed above) combined with existing belief `es-persistence-uses-jsonl`
- Source: entries/2026/05/29/topic-snapshot-storage-format.md

### [REJECT] btree-wal-is-redo-only
Duplicates existing belief `btree-wal-is-physical-redo-only`
- Source: entries/2026/05/29/topic-split-atomicity-via-wal.md

### [ACCEPT] btree-wal-fsync-per-entry
Each `WAL.log_write()` call performs an `fsync`, guaranteeing that WAL entries are durable before any data page writes proceed
- Source: entries/2026/05/29/topic-split-atomicity-via-wal.md

### [REJECT] btree-page-writes-not-individually-durable
Duplicates existing belief `btree-wal-no-data-fsync` — both capture that `PageManager.write_page()` does not fsync
- Source: entries/2026/05/29/topic-split-atomicity-via-wal.md

### [ACCEPT] btree-wal-truncation-is-commit-point
The WAL truncation in `commit()` is the atomic linearization point: before truncation, recovery will redo the transaction; after truncation, the transaction is committed
- Source: entries/2026/05/29/topic-split-atomicity-via-wal.md

### [REJECT] btree-has-separate-wal-from-standalone
Duplicates existing belief `two-wal-designs-in-repo`
- Source: entries/2026/05/29/topic-split-atomicity-via-wal.md


## Entry 1: `topic-sstable-block-format.md`

### [ACCEPT] sstable-lookup-is-two-phase
Point lookups in both SSTable implementations first binary-search the sparse index to identify a block, then linearly scan entries within that block's byte range; total cost is O(log(N/B) + B)
- Source: entries/2026/05/29/topic-sstable-block-format.md

### [ACCEPT] test-block-size-differs-from-default
Tests universally use `block_size=4` while `SSTableWriter` defaults to 64 and `lsm.py` defaults to 16; the small test value forces multi-block index paths with small datasets
- Source: entries/2026/05/29/topic-sstable-block-format.md

### [ACCEPT] two-sstable-implementations-same-pattern
`lsm.py` (using `sparse_index_interval=16` and `bisect`) and `sstable.py` (using `block_size=64` and manual binary search) implement the same sparse-index-with-block-scan pattern with different defaults, naming, and file format maturity
- Source: entries/2026/05/29/topic-sstable-block-format.md

### [REJECT] block-size-controls-index-density
Duplicates existing belief `sstable-sparse-index-every-nth` which already captures that every Nth entry is recorded in the sparse index
- Source: entries/2026/05/29/topic-sstable-block-format.md

---

## Entry 2: `topic-tester-runner-system.md`

### [ACCEPT] no-ci-or-runner-for-tester-tests
No CI pipeline (no `.github/` directory), Makefile, or orchestration script exists to invoke `tester_test_*.py` files; they are run manually via `python tester_test_*.py`
- Source: entries/2026/05/29/topic-tester-runner-system.md

### [ACCEPT] tester-files-are-generated-artifacts
The `tester_test_*.py` files are generated by the code-expert workflow's tester stage from implementation specs, distinct from hand-written `test_*.py` pytest files
- Source: entries/2026/05/29/topic-tester-runner-system.md

### [REJECT] tester-output-is-human-readable
Duplicates existing belief `tester-files-use-stdout-validation` which already captures the stdout PASSED/FAILED reporting convention
- Source: entries/2026/05/29/topic-tester-runner-system.md

### [REJECT] tester-tests-are-zero-dependency
Duplicates existing belief `tester-files-are-standalone-runnable` which already captures that these files run independently of any test framework
- Source: entries/2026/05/29/topic-tester-runner-system.md

### [REJECT] assert-abort-is-failure-signal
Trivial Python runtime behavior — `assert` raising and halting execution is not a codebase-specific invariant
- Source: entries/2026/05/29/topic-tester-runner-system.md

---

## Entry 3: `topic-tester-test-pattern.md`

### [ACCEPT] naming-convention-not-uniform
The tester/test split uses at least three naming patterns across directories: `tester_test_*.py`, `test_*_tester.py`, and `test_tester_*.py`
- Source: entries/2026/05/29/topic-tester-test-pattern.md

### [ACCEPT] both-suites-discoverable-by-pytest
Both `tester_test_*.py` and `test_*.py` naming conventions match pytest's default collection patterns (`test_*.py` and `*_test_*.py`), so `pytest` in any project directory runs the combined suite — **note: this contradicts existing belief `tester-naming-avoids-pytest-discovery`; one should be retracted**
- Source: entries/2026/05/29/topic-tester-test-pattern.md

### [REJECT] tester-files-are-spec-verification
Duplicates existing beliefs `tester-tests-public-contract` (tester files test only the public API) and `pytest-files-have-broader-coverage` (pytest files cover more including internals)
- Source: entries/2026/05/29/topic-tester-test-pattern.md

### [REJECT] tester-files-never-import-internals
Duplicates existing belief `tester-tests-public-contract` which already captures that tester files use only the public interface
- Source: entries/2026/05/29/topic-tester-test-pattern.md

### [REJECT] tester-files-run-standalone
Duplicates existing belief `tester-files-are-standalone-runnable` which already captures the `__main__` block / independent execution
- Source: entries/2026/05/29/topic-tester-test-pattern.md

---

## Entry 4: `topic-tombstone-handling-in-hint-files.md`

### [ACCEPT] lsht-hint-no-tombstone-flag
The `log-structured-hash-table` hint entry format (`!II` = key_size + offset) has no field to distinguish live records from tombstones, making `_load_hint_file` structurally unable to replicate `_scan_segment`'s deletion logic
- Source: entries/2026/05/29/topic-tombstone-handling-in-hint-files.md

### [ACCEPT] lsht-get-no-tombstone-guard
`BitcaskStore.get()` in `log-structured-hash-table/bitcask.py` returns raw payload bytes without checking for the `TOMBSTONE` sentinel, so an index entry pointing to a tombstone record surfaces `b"__BITCASK_TOMBSTONE__"` as a value
- Source: entries/2026/05/29/topic-tombstone-handling-in-hint-files.md

### [ACCEPT] hash-index-get-tombstone-guard
`hash-index-storage/bitcask.py`'s `get()` checks `if value == "": return None`, providing a defense-in-depth layer against tombstone leaks that `log-structured-hash-table`'s `get()` lacks
- Source: entries/2026/05/29/topic-tombstone-handling-in-hint-files.md

### [REJECT] lsht-recovery-path-divergence
Duplicates existing belief `bitcask-tombstone-handling-asymmetry` which already captures the divergent behavior between hint-based and scan-based recovery paths
- Source: entries/2026/05/29/topic-tombstone-handling-in-hint-files.md

---

## Entry 5: `topic-tombstone-semantics.md`

### [ACCEPT] log-structured-sentinel-is-in-band
`log-structured-hash-table/bitcask.py` uses `TOMBSTONE = b"__BITCASK_TOMBSTONE__"` as an in-band sentinel in the value space; storing those exact bytes as a value causes silent data loss on recovery when the record is misinterpreted as a deletion
- Source: entries/2026/05/29/topic-tombstone-semantics.md

### [REJECT] hash-index-empty-string-is-tombstone
Duplicates existing belief `bitcask-tombstone-is-empty-string` which already captures that empty-string values are indistinguishable from deletions
- Source: entries/2026/05/29/topic-tombstone-semantics.md

### [REJECT] hash-index-no-crc
Duplicates existing belief `bitcask-no-checksum-validation` which already captures the absence of integrity checking in `hash-index-storage`
- Source: entries/2026/05/29/topic-tombstone-semantics.md

### [REJECT] header-flag-zero-value-collision
Design recommendation rather than a testable factual claim about the code — "neither implementation uses X" is better expressed through the existing per-implementation beliefs about their actual tombstone encoding
- Source: entries/2026/05/29/topic-tombstone-semantics.md


### [ACCEPT] codebase-uses-no-steal-policy
All implementations use NO-STEAL buffer management: uncommitted transaction data is never written to the persistent data store, eliminating the need for undo logging at the cost of requiring all dirty data from active transactions to fit in memory
- Source: entries/2026/05/29/topic-undo-logging-and-steal-policy.md

### [ACCEPT] wal-has-no-before-images
WALRecord stores only `key` and `value` (the new value) with no `old_value` field, making undo structurally impossible from the log alone
- Source: entries/2026/05/29/topic-undo-logging-and-steal-policy.md

### [ACCEPT] mvcc-uncommitted-data-memory-only
In MVCCDatabase, uncommitted transaction writes exist only as in-memory Version objects in `_versions` and never reach a persistent data store; abort works by adding the tx_id to `_aborted` so visibility checks filter it out
- Source: entries/2026/05/29/topic-undo-logging-and-steal-policy.md

### [ACCEPT] abort-is-status-change-not-disk-rollback
Aborting a transaction in both MVCC and SSI implementations sets a status flag (`_aborted` set or status marker); no disk writes are reversed because uncommitted data never reached disk
- Source: entries/2026/05/29/topic-undo-logging-and-steal-policy.md

### [REJECT] btree-wal-is-redo-only
Duplicates existing belief `btree-wal-is-physical-redo-only`
- Source: entries/2026/05/29/topic-undo-logging-and-steal-policy.md

### [ACCEPT] write-meta-no-fsync
`PageManager._write_meta` calls `flush()` but never `os.fsync()`, so metadata updates to page 0 are not guaranteed durable even when they appear to have been written; the `sync()` method does call `os.fsync()` but nothing in the allocation path invokes it
- Source: entries/2026/05/29/topic-wal-atomicity-gap.md

### [REJECT] allocate-page-bypasses-wal
Duplicates existing beliefs `btree-allocate-page-outside-wal` and `btree-metadata-bypasses-wal`
- Source: entries/2026/05/29/topic-wal-atomicity-gap.md

### [REJECT] wal-is-redo-only-for-page-data
Duplicates existing beliefs `btree-wal-is-physical-redo-only` and `wal-protects-data-not-metadata`
- Source: entries/2026/05/29/topic-wal-atomicity-gap.md

### [REJECT] free-page-same-vulnerability
Duplicates existing belief `btree-free-page-bypasses-wal`
- Source: entries/2026/05/29/topic-wal-atomicity-gap.md

### [ACCEPT] wal-docstring-describes-intent-not-behavior
The `replay()` docstring claims it "skips uncommitted batches" (describing the DDIA concept), but the inline comments and implementation show it returns all PUT/DELETE records regardless of COMMIT presence; the docstring is aspirational, the comments are accurate
- Source: entries/2026/05/29/topic-wal-batch-atomicity-gap.md

### [REJECT] iterate-enables-correct-replay
Duplicates existing belief `wal-iterate-preserves-all-ops`
- Source: entries/2026/05/29/topic-wal-batch-atomicity-gap.md

### [REJECT] batch-atomicity-is-fsync-level-only
Duplicates existing belief `wal-batch-relies-on-practical-atomicity`
- Source: entries/2026/05/29/topic-wal-batch-atomicity-gap.md

### [REJECT] wal-tail-corruption-silent-truncation
Duplicates existing beliefs `wal-partial-read-is-eof` and `wal-corruption-returns-valid-prefix`
- Source: entries/2026/05/29/topic-wal-corruption-models.md

### [REJECT] wal-mid-corruption-stops-replay
Duplicates existing beliefs `wal-crc-mismatch-halts-all-replay` and `wal-no-resync-after-corruption`
- Source: entries/2026/05/29/topic-wal-corruption-models.md

### [REJECT] wal-crc-excludes-length-and-seqnum
Duplicates existing beliefs `wal-crc-does-not-cover-seqnum` and `all-three-implementations-use-payload-only-crc`
- Source: entries/2026/05/29/topic-wal-corruption-models.md

### [REJECT] bitcask-has-no-integrity-checks
Contradicts existing beliefs `bitcask-crc-covers-payload-only` and `bitcask-crc32-raises-corruption-error` which indicate Bitcask does have CRC checksums
- Source: entries/2026/05/29/topic-wal-corruption-models.md

### [ACCEPT] lsm-wal-has-no-checksums
The LSM tree's WAL (`lsm.py:13-63`) uses length-prefixed records with no CRC, relying solely on short-read detection for crash recovery; distinct from the separate `lsm-wal-has-no-fsync` issue
- Source: entries/2026/05/29/topic-wal-corruption-models.md

### [REJECT] wal-seq-num-monotonic
Duplicates existing belief `wal-seq-nums-strictly-monotonic`
- Source: entries/2026/05/29/topic-wal-crash-recovery-semantics.md

### [REJECT] wal-crc-guards-replay
Duplicates existing belief `wal-crc-mismatch-halts-all-replay`
- Source: entries/2026/05/29/topic-wal-crash-recovery-semantics.md

### [ACCEPT] wal-append-mode-no-overwrite
`_open_latest` opens WAL files in `"ab"` (append-binary) mode, making it impossible for post-crash writes to overwrite pre-crash records; this is the bridge between recovery and forward progress
- Source: entries/2026/05/29/topic-wal-crash-recovery-semantics.md

### [REJECT] wal-batch-commit-always-fsynced
Duplicates existing belief `wal-batch-always-fsyncs`
- Source: entries/2026/05/29/topic-wal-crash-recovery-semantics.md

### [REJECT] wal-file-ordering-invariant
Duplicates existing beliefs `wal-files-sorted-lexicographic` and `wal-segment-naming-zero-padded`
- Source: entries/2026/05/29/topic-wal-crash-recovery-semantics.md


### [ACCEPT] wal-default-segment-10mb
WAL and hash-index Bitcask both default `max_file_size` to 10 MB, while the log-structured Bitcask defaults to 1 MB — reflecting the latter's heavier reliance on compaction with hint files to offset recovery cost.
- Source: entries/2026/05/29/topic-wal-segment-sizing-tradeoffs.md

### [ACCEPT] recovery-scans-all-segments
All three storage implementations (WAL, hash-index Bitcask, log-structured Bitcask) scan every segment file on startup to rebuild state, making recovery time proportional to segment count rather than data volume.
- Source: entries/2026/05/29/topic-wal-segment-sizing-tradeoffs.md

### [ACCEPT] compaction-respects-size-limit
Bitcask compaction output is itself subject to `max_file_size` rotation, so smaller size limits produce more post-compaction files — feeding back into recovery cost.
- Source: entries/2026/05/29/topic-wal-segment-sizing-tradeoffs.md

### [REJECT] auto-compact-triggers-on-frozen-count
Duplicates existing belief `bitcask-auto-compact-threshold`.
- Source: entries/2026/05/29/topic-wal-segment-sizing-tradeoffs.md

### [REJECT] hint-files-skip-value-scanning
Duplicates existing belief `bitcask-hint-files-skip-scan`.
- Source: entries/2026/05/29/topic-wal-segment-sizing-tradeoffs.md

### [ACCEPT] wal-sync-mode-default-is-sync
`WriteAheadLog` defaults to `sync_mode="sync"`, calling `flush()` + `os.fsync()` after every single `append` call — the safest and slowest mode.
- Source: entries/2026/05/29/topic-wal-sync-modes.md

### [REJECT] wal-batch-force-sync
Duplicates existing beliefs `force-true-bypasses-sync-mode` and `wal-batch-always-fsyncs`.
- Source: entries/2026/05/29/topic-wal-sync-modes.md

### [REJECT] wal-batch-counter-reset
Misleading given the known bug documented in `batch-write-count-not-reset-on-forced-sync` — the counter does NOT always reflect actual unsynced writes after forced syncs.
- Source: entries/2026/05/29/topic-wal-sync-modes.md

### [REJECT] wal-none-mode-no-sync
Duplicates existing belief `none-mode-never-syncs-on-append`.
- Source: entries/2026/05/29/topic-wal-sync-modes.md

### [REJECT] wal-crc-detects-torn-writes
Duplicates existing belief `crc-detects-not-prevents-data-loss`.
- Source: entries/2026/05/29/topic-wal-sync-modes.md

### [REJECT] wal-write-count-not-reset-on-forced-sync
Duplicates existing belief `batch-write-count-not-reset-on-forced-sync`.
- Source: entries/2026/05/29/topic-write-count-reset-semantics.md

### [ACCEPT] wal-do-sync-branches-mutually-exclusive
The `_do_sync` method's `if`/`elif` structure means forced syncs in batch mode skip all counter logic entirely — the counter is neither incremented nor reset, which is the structural root cause of the premature batch-sync bug.
- Source: entries/2026/05/29/topic-write-count-reset-semantics.md

### [REJECT] wal-rotate-bypasses-do-sync
Overlaps with existing belief `rotation-always-fsyncs`; the counter-reset implication is a second-order effect of the broader issue already captured in `batch-write-count-not-reset-on-forced-sync`.
- Source: entries/2026/05/29/topic-write-count-reset-semantics.md

### [ACCEPT] wal-batch-bug-is-performance-not-correctness
The stale `_write_count` after forced syncs causes extra fsyncs (premature batch threshold triggers) but never causes data loss — forced syncs always flush to disk regardless of counter state.
- Source: entries/2026/05/29/topic-write-count-reset-semantics.md

### [ACCEPT] unbundled-db-put-returns-cdc-event
`UnbundledDatabase.put()` and `delete()` return `CDCEvent` objects directly, making CDC a synchronous, first-class part of the write API rather than a side-channel.
- Source: entries/2026/05/29/unbundled-database-test_unbundled_database.md

### [ACCEPT] unbundled-db-flush-required-for-derived
Derived systems (secondary indexes, materialized views, full-text search) only see mutations after `db.flush()` is called; writes go to WAL and storage immediately but CDC consumers are decoupled.
- Source: entries/2026/05/29/unbundled-database-test_unbundled_database.md

### [ACCEPT] unbundled-db-rebuild-equals-live
`rebuild_system()` must produce state identical to incremental live processing — this is a tested invariant verified by capturing `get_state()` before and after rebuild.
- Source: entries/2026/05/29/unbundled-database-test_unbundled_database.md

### [ACCEPT] unbundled-db-catch-up-replays-history
A derived system added with `catch_up=True` receives all historical CDC events to reach current state, without requiring a separate snapshot mechanism.
- Source: entries/2026/05/29/unbundled-database-test_unbundled_database.md

### [ACCEPT] unbundled-db-wal-persistence-jsonl
The unbundled database's WAL supports optional file-backed persistence via a `.jsonl` file path, and a new `WriteAheadLog` instance recovers entries from that file on init.
- Source: entries/2026/05/29/unbundled-database-test_unbundled_database.md

### [REJECT] unbundled-db-flush-is-consistency-boundary
Duplicates `unbundled-db-flush-required-for-derived` proposed above from the companion test file.
- Source: entries/2026/05/29/unbundled-database-tester_test_unbundled_database.md

### [ACCEPT] unbundled-db-cdc-events-carry-old-value
Every update and delete CDC event includes the previous value (`old_value`), enabling derived systems to undo prior state; inserts have `old_value=None`.
- Source: entries/2026/05/29/unbundled-database-tester_test_unbundled_database.md

### [REJECT] unbundled-db-rebuild-is-idempotent
Duplicates `unbundled-db-rebuild-equals-live` proposed above from the companion test file.
- Source: entries/2026/05/29/unbundled-database-tester_test_unbundled_database.md

### [REJECT] unbundled-db-catch-up-replays-full-log
Duplicates `unbundled-db-catch-up-replays-history` proposed above from the companion test file.
- Source: entries/2026/05/29/unbundled-database-tester_test_unbundled_database.md

### [ACCEPT] unbundled-db-wal-lsn-starts-at-one
The unbundled database WAL assigns LSNs starting from 1 (not 0), with `latest_lsn == 0` indicating an empty log.
- Source: entries/2026/05/29/unbundled-database-tester_test_unbundled_database.md


### [ACCEPT] vector-clock-immutability
`VectorClock` is immutable: `increment`, `merge`, and `prune` all return new instances and never modify `_clock` in place.
- Source: entries/2026/05/29/vector-clocks-vector_clock.md

### [ACCEPT] sibling-preservation-on-concurrent-writes
`VersionedKVStore.put` never drops an existing version unless the new version's clock dominates it; concurrent versions are preserved as siblings.
- Source: entries/2026/05/29/vector-clocks-vector_clock.md

### [ACCEPT] reconcile-replaces-all-siblings
`reconcile` unconditionally replaces all siblings for a key with a single merged version, regardless of whether the provided contexts cover every existing sibling.
- Source: entries/2026/05/29/vector-clocks-vector_clock.md

### [ACCEPT] zero-entries-stripped
`VectorClock.__init__` strips all entries with value 0, ensuring two clocks with identical non-zero entries are always equal and hash-equal.
- Source: entries/2026/05/29/vector-clocks-vector_clock.md

### [ACCEPT] compare-returns-four-states
`VectorClock.compare` returns exactly one of `BEFORE`, `AFTER`, `EQUAL`, or `CONCURRENT`, implementing the partial order defined by component-wise comparison.
- Source: entries/2026/05/29/vector-clocks-vector_clock.md

### [ACCEPT] wal-sync-mode-unvalidated
`WriteAheadLog` accepts any string as `sync_mode` without validation; values other than `"sync"` and `"batch"` silently disable all fsync.
- Source: entries/2026/05/29/write-ahead-log-wal-_do_sync.md

### [REJECT] wal-force-sync-skips-batch-counter-reset
Duplicates existing belief `batch-write-count-not-reset-on-forced-sync`.
- Source: entries/2026/05/29/write-ahead-log-wal-_do_sync.md

### [ACCEPT] wal-batch-sync-count-controls-durability-window
In batch mode, up to `batch_sync_count - 1` records may be lost on crash because they haven't been fsynced yet.
- Source: entries/2026/05/29/write-ahead-log-wal-_do_sync.md

### [ACCEPT] wal-do-sync-requires-caller-lock
`_do_sync` does not acquire `self._lock`; thread safety depends on every caller holding the lock before invoking it.
- Source: entries/2026/05/29/write-ahead-log-wal-_do_sync.md

### [ACCEPT] wal-rotation-is-post-write
Rotation is checked after each write is synced, never before; a single write can produce a segment larger than `_max_file_size`.
- Source: entries/2026/05/29/write-ahead-log-wal-_maybe_rotate.md

### [ACCEPT] wal-max-file-size-is-soft-limit
`_max_file_size` is a soft cap: the check triggers rotation for the *next* write, it does not prevent the current write from exceeding the limit.
- Source: entries/2026/05/29/write-ahead-log-wal-_maybe_rotate.md

### [ACCEPT] maybe-rotate-requires-lock
`_maybe_rotate` must be called under `self._lock` but does not acquire or assert the lock itself; all three call sites (`append`, `append_batch`, `checkpoint`) satisfy this obligation.
- Source: entries/2026/05/29/write-ahead-log-wal-_maybe_rotate.md

### [ACCEPT] rotation-preserves-fd-invariant
After `_maybe_rotate` returns, `self._fd` is guaranteed non-None and open for writing (assuming it was non-None on entry), because `_rotate()` always opens a new file.
- Source: entries/2026/05/29/write-ahead-log-wal-_maybe_rotate.md

### [ACCEPT] wal-size-check-uses-disk
`_open_latest` checks on-disk file size via `os.path.getsize`, not an in-memory counter, making it correct across crash/restart boundaries.
- Source: entries/2026/05/29/write-ahead-log-wal-_open_latest.md

### [ACCEPT] wal-exactly-at-limit-rotates
A WAL file whose size equals `_max_file_size` triggers rotation in `_open_latest`; only files strictly smaller are reused.
- Source: entries/2026/05/29/write-ahead-log-wal-_open_latest.md

### [REJECT] wal-open-latest-postcondition
Substantially overlaps with `rotation-preserves-fd-invariant`; same postcondition pattern for a closely related method.
- Source: entries/2026/05/29/write-ahead-log-wal-_open_latest.md

### [REJECT] wal-open-latest-two-callers
Call site enumeration is unstable and more trivia than invariant; not something that would break if violated.
- Source: entries/2026/05/29/write-ahead-log-wal-_open_latest.md

### [REJECT] wal-append-sequence-gap-free
Duplicates existing belief `wal-seq-nums-strictly-monotonic`.
- Source: entries/2026/05/29/write-ahead-log-wal-append.md

### [ACCEPT] wal-append-not-transactional
Individual `append` calls are not wrapped in any transaction boundary; for atomic multi-operation writes, `append_batch` must be used instead.
- Source: entries/2026/05/29/write-ahead-log-wal-append.md

### [REJECT] wal-sync-mode-durability
Covered by existing beliefs `wal-sync-mode-survives-crash` and `batch-mode-defers-fsync`.
- Source: entries/2026/05/29/write-ahead-log-wal-append.md

### [ACCEPT] wal-append-no-validation
`append` performs no validation of `op_type` against `OP_BYTES`; invalid operation names raise `KeyError` from the dictionary lookup, not a descriptive error.
- Source: entries/2026/05/29/write-ahead-log-wal-append.md

### [REJECT] wal-rotation-on-size
The core claim (WAL rotates based on file size) is obvious from the code structure; the more interesting ordering invariant is captured by `wal-rotation-is-post-write`.
- Source: entries/2026/05/29/write-ahead-log-wal-append.md


### [REJECT] checkpoint-force-syncs
Duplicate of existing `force-true-bypasses-sync-mode` which already captures that `force=True` unconditionally syncs regardless of sync mode setting
- Source: entries/2026/05/29/write-ahead-log-wal-checkpoint.md

### [REJECT] checkpoint-shares-sequence-space
Duplicate of existing `wal-checkpoint-consumes-sequence` which already captures that checkpoint records consume from the shared monotonic sequence counter
- Source: entries/2026/05/29/write-ahead-log-wal-checkpoint.md

### [ACCEPT] checkpoint-empty-payload
Checkpoint records are encoded with zero-length key and value fields; the record's semantic role is carried entirely by its `OP_CHECKPOINT` type and sequence number
- Source: entries/2026/05/29/write-ahead-log-wal-checkpoint.md

### [ACCEPT] checkpoint-filtered-from-replay
`replay()` filters out CHECKPOINT records (along with COMMIT records), returning only PUT and DELETE — checkpoint records serve as truncation boundaries, not data operations visible to recovery consumers
- Source: entries/2026/05/29/write-ahead-log-wal-checkpoint.md


