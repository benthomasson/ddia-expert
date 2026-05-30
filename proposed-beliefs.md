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



---

**Generated:** 2026-05-29
**Source:** 235 entries from entries/
**Model:** claude

## Entry: avro-serializer-avro_serializer-_resolve_record.md

### [ACCEPT] resolve-record-two-pass
`_resolve_record` uses a two-pass algorithm: pass 1 consumes all writer fields from the buffer in wire order (decoding or skipping each), pass 2 assembles the result dict in reader field order from decoded values and defaults
- Source: entries/2026/05/29/avro-serializer-avro_serializer-_resolve_record.md

### [REJECT] resolve-record-default-fill
Duplicates existing belief `avro-missing-reader-field-requires-default`
- Source: entries/2026/05/29/avro-serializer-avro_serializer-_resolve_record.md

### [ACCEPT] resolve-record-skip-unknown
Writer fields not present in the reader schema are fully consumed from the buffer via `_skip()` and discarded, ensuring the byte stream stays correctly positioned for subsequent fields
- Source: entries/2026/05/29/avro-serializer-avro_serializer-_resolve_record.md

### [ACCEPT] resolve-record-dead-code
`writer_field_map` is constructed but never referenced in `_resolve_record`, making it dead code likely left from a refactor
- Source: entries/2026/05/29/avro-serializer-avro_serializer-_resolve_record.md

### [ACCEPT] resolve-record-shared-defaults
Default values in `_resolve_record` are inserted by reference without deep copying, so mutable defaults (lists, dicts) would be shared across all decoded records that use the default
- Source: entries/2026/05/29/avro-serializer-avro_serializer-_resolve_record.md

---

## Entry: avro-serializer-test_avro.md

### [REJECT] avro-no-field-names-on-wire
Duplicates existing belief `avro-binary-no-field-names`
- Source: entries/2026/05/29/avro-serializer-test_avro.md

### [REJECT] avro-backward-compat-requires-defaults
Duplicates existing belief `avro-missing-reader-field-requires-default`
- Source: entries/2026/05/29/avro-serializer-test_avro.md

### [REJECT] avro-decoder-two-schema-resolution
Duplicates existing belief `avro-writer-reader-dual-schema-resolution`
- Source: entries/2026/05/29/avro-serializer-test_avro.md

### [REJECT] avro-type-promotion-widening-only
Duplicates existing belief `avro-promotions-one-directional`
- Source: entries/2026/05/29/avro-serializer-test_avro.md

### [ACCEPT] avro-enum-default-fallback
When a writer sends an enum symbol not in the reader's symbol list, the reader falls back to the `default` symbol declared in the reader schema rather than raising an error
- Source: entries/2026/05/29/avro-serializer-test_avro.md

---

## Entry: avro-serializer-test_avro_serializer.md

### [REJECT] avro-decoder-dual-schema
Duplicates existing belief `avro-writer-reader-dual-schema-resolution`
- Source: entries/2026/05/29/avro-serializer-test_avro_serializer.md

### [REJECT] avro-binary-no-field-names
Already exists as an accepted belief
- Source: entries/2026/05/29/avro-serializer-test_avro_serializer.md

### [ACCEPT] avro-schema-canonical-equality
`Schema("int")` and `Schema({"type": "int"})` are equal and hash-equal, enabling consistent set/dict usage regardless of whether the schema was constructed from a string or dict form
- Source: entries/2026/05/29/avro-serializer-test_avro_serializer.md

### [ACCEPT] avro-union-disambiguation
When encoding a Python dict into a union containing both a record and a map type, the encoder resolves ambiguity by matching dict keys against record field names first, falling back to map only if no record matches
- Source: entries/2026/05/29/avro-serializer-test_avro_serializer.md

### [ACCEPT] avro-int-bounds-enforced
The Avro `int` type enforces 32-bit signed integer range ([-2^31, 2^31-1]) at encode time via `ValueError`, while `long` accepts values at least as large as 2^40
- Source: entries/2026/05/29/avro-serializer-test_avro_serializer.md

---

## Entry: b-tree-storage-engine-btree-PageManager.md

### [ACCEPT] page-manager-flush-not-durable
PageManager calls `flush()` on every `write_page` and `_write_meta` call but only calls `os.fsync()` in `sync()` and `close()`, so individual page writes are not crash-durable without the WAL layer
- Source: entries/2026/05/29/b-tree-storage-engine-btree-PageManager.md

### [REJECT] page-manager-free-list-is-intrusive
Duplicates existing belief `btree-free-list-intrusive`
- Source: entries/2026/05/29/b-tree-storage-engine-btree-PageManager.md

### [ACCEPT] page-manager-no-page-size-on-disk
The page size is not persisted in the data file; reopening with a different `page_size` value silently corrupts all page reads with no error or detection mechanism
- Source: entries/2026/05/29/b-tree-storage-engine-btree-PageManager.md

### [REJECT] page-manager-allocate-prefers-free-list
Duplicates existing belief `allocate-page-prefers-free-list`
- Source: entries/2026/05/29/b-tree-storage-engine-btree-PageManager.md

### [ACCEPT] page-manager-meta-reads-untracked
`read_meta()` does not increment `pages_read`, so metadata access is invisible to the I/O statistics reported by `BTree.stats`
- Source: entries/2026/05/29/b-tree-storage-engine-btree-PageManager.md

### [ACCEPT] page-zero-is-metadata
Page 0 is always the metadata page and must never be written directly — only through `_write_meta`; user data pages start at page 1, with the initial empty leaf root always at page 1
- Source: entries/2026/05/29/b-tree-storage-engine-btree-PageManager.md

---

## Entry: b-tree-storage-engine-btree-_checksum.md

### [REJECT] wal-checksum-is-crc32
Already covered by existing beliefs including `btree-wal-uses-trailer-crc` and `all-three-implementations-use-payload-only-crc`
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_checksum.md

### [REJECT] wal-checksum-covers-data-only
Duplicates existing belief `btree-wal-checksum-excludes-page-num`
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_checksum.md

### [REJECT] wal-recovery-skips-bad-checksum
Duplicates existing belief `btree-crc32-skips-corrupt-entries`
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_checksum.md

### [ACCEPT] checksum-mask-is-python2-compat
The `& 0xFFFFFFFF` mask in `WAL._checksum` is a Python 2 portability idiom — Python 3's `zlib.crc32` already returns unsigned values, making the mask technically redundant but kept as defensive code
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_checksum.md


### [REJECT] internal-node-children-keys-invariant
Duplicates existing `internal-node-children-invariant`
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_deserialize_internal.md

### [REJECT] internal-deserialize-no-type-check
Covered by existing `btree-deserialize-no-validation` which already states neither deserializer validates page type
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_deserialize_internal.md

### [REJECT] internal-node-binary-layout
Duplicates existing `internal-page-binary-layout`
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_deserialize_internal.md

### [REJECT] deserialize-internal-pure-function
Too trivial — a deserialization function having no side effects is expected, not an invariant that would break things if violated
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_deserialize_internal.md

### [REJECT] internal-keys-utf8-assumption
Covered by existing `btree-keys-decoded-as-utf8` which already captures that keys are decoded as UTF-8
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_deserialize_internal.md

### [ACCEPT] btree-bisect-left-leaf-bisect-right-internal
`_search` uses `bisect_left` at leaf nodes for exact-match lookup and `bisect_right` at internal nodes for child routing; this asymmetry matches the split convention where the median key is copied up and retained as the first key of the right leaf.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_search.md

### [REJECT] btree-search-reads-exactly-height-pages
Covered by existing `btree-lookup-reads-bounded-by-height`
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_search.md

### [REJECT] btree-search-is-read-only
Too trivial — a search/get function being read-only is the expected default
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_search.md

### [ACCEPT] btree-depth-param-not-validated
`_search` trusts that the `depth` parameter from metadata accurately reflects the tree's balance; corrupted height causes silent wrong-format deserialization (leaf data parsed as internal or vice versa) with no error.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_search.md

### [ACCEPT] meta-page-is-page-zero
The B-tree metadata (root, height, total_keys, next_free, free_head) is always stored at file offset 0 in page 0, which is reserved as the superblock and occupies exactly `page_size` bytes.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_write_meta.md

### [REJECT] meta-flush-without-fsync
Duplicates existing `write-meta-no-fsync`
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_write_meta.md

### [ACCEPT] meta-page-fixed-20-byte-payload
The metadata payload is exactly 20 bytes (5 big-endian uint32s packed as `'>5I'`), zero-padded to `page_size`; the format requires `page_size >= 20` but this is not enforced.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_write_meta.md

### [ACCEPT] init-meta-unprotected-by-wal
During initial file creation, `PageManager.__init__` writes metadata and the root leaf page directly via `_write_meta` without WAL protection; acceptable because no user data exists yet to lose.
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_write_meta.md

### [REJECT] allocate-free-bypass-wal
Covered by existing `btree-allocate-page-outside-wal` and `btree-free-page-bypasses-wal`
- Source: entries/2026/05/29/b-tree-storage-engine-btree-_write_meta.md

### [ACCEPT] pipeline-records-are-tuples
Pipeline stages produce and consume lists of `(key, value)` tuples, where key is a string and value depends on the stage (string for Tokenize, int for Count).
- Source: entries/2026/05/29/batch-word-count-test_pipeline.md

### [ACCEPT] readlines-accepts-list-or-filepath
`ReadLines` accepts either a `list[str]` of in-memory lines or a file path string as its input source, providing two input modes from the same stage.
- Source: entries/2026/05/29/batch-word-count-test_pipeline.md

### [ACCEPT] pipeline-run-returns-list
`Pipeline.run()` materializes all output records into a `list`, while `run_lazy()` returns an iterator that yields records on demand without full materialization.
- Source: entries/2026/05/29/batch-word-count-test_pipeline.md

### [ACCEPT] external-sort-is-transparent
`Sort` with `memory_limit` produces the same output ordering as an in-memory sort — the external merge-sort is a pure implementation optimization with no semantic difference.
- Source: entries/2026/05/29/batch-word-count-test_pipeline.md

### [REJECT] pipeline-stats-tracks-all-stages
Minor feature detail rather than an invariant; violating it would not break pipeline correctness
- Source: entries/2026/05/29/batch-word-count-test_pipeline.md

### [ACCEPT] bloom-filter-no-false-negatives
After `BloomFilter.add(x)`, `x in filter` always returns `True`; the data structure guarantees zero false negatives by design (all k bit positions are set on add and all must be set for membership).
- Source: entries/2026/05/29/bloom-filter-bloom_filter-BloomFilter.md

### [ACCEPT] bloom-filter-double-hashing
`BloomFilter` generates k bit positions via Kirschner-Mitzenmacker double hashing: one MD5 digest is split into two 64-bit values h1 and h2, then positions are computed as `(h1 + i*h2) % m` rather than using k independent hash functions.
- Source: entries/2026/05/29/bloom-filter-bloom_filter-BloomFilter.md

### [ACCEPT] bloom-filter-monotonic-bits
Bits in `BloomFilter` are only ever set, never cleared; this monotonic property makes the structure append-only, enables union via bitwise OR, and prevents deletion without a counting layer.
- Source: entries/2026/05/29/bloom-filter-bloom_filter-BloomFilter.md

### [REJECT] bloom-filter-count-is-insertions
Covered in spirit by existing `cbf-count-tracks-calls-not-cardinality` — the base `BloomFilter` shares identical `_count` behavior since `CountingBloomFilter` inherits it
- Source: entries/2026/05/29/bloom-filter-bloom_filter-BloomFilter.md

### [ACCEPT] bloom-filter-union-requires-identical-params
`BloomFilter.union()` raises `ValueError` if the two filters differ in bit array size (m) or hash count (k), since bitwise OR is only meaningful when bit positions map to the same hash space.
- Source: entries/2026/05/29/bloom-filter-bloom_filter-BloomFilter.md


### [REJECT] bloom-double-hashing-two-from-one
Duplicate of existing `bloom-double-hashing` which already captures the Kirsch–Mitzenmacher double hashing scheme
- Source: entries/2026/05/29/bloom-filter-bloom_filter-_hashes.md

### [REJECT] bloom-hashes-pure-deterministic
Subsumed by `bloom-filter-deterministic-hashing` (proposed below), which captures the same determinism property at the filter level
- Source: entries/2026/05/29/bloom-filter-bloom_filter-_hashes.md

### [ACCEPT] bloom-hashes-positions-not-unique
The `k` positions returned by `_hashes` are not guaranteed distinct — duplicate positions can occur when `m` is small relative to `k`, and all callers (add, contains, remove) tolerate this via idempotent bit/counter operations
- Source: entries/2026/05/29/bloom-filter-bloom_filter-_hashes.md

### [ACCEPT] bloom-md5-split-64bit
`_hashes` splits the 128-bit MD5 digest at the midpoint into two 64-bit little-endian unsigned integers `h1` and `h2`, used as base position and step size for double hashing
- Source: entries/2026/05/29/bloom-filter-bloom_filter-_hashes.md

### [ACCEPT] bloom-hashes-no-m-zero-guard
`_hashes` does not guard against `m=0`, which causes an unhandled `ZeroDivisionError` in the `(h1 + i * h2) % m` computation
- Source: entries/2026/05/29/bloom-filter-bloom_filter-_hashes.md

### [ACCEPT] cbf-remove-two-pass-atomicity
`CountingBloomFilter.remove()` validates all hash positions have non-zero counters in a first pass before decrementing any in a second pass, preventing partial state corruption when removing an absent item
- Source: entries/2026/05/29/bloom-filter-bloom_filter-remove.md

### [REJECT] cbf-saturated-counters-never-decremented
Duplicate of existing `cbf-saturated-counters-never-decrement`
- Source: entries/2026/05/29/bloom-filter-bloom_filter-remove.md

### [REJECT] cbf-remove-false-positive-acceptance
Duplicate of existing `cbf-remove-false-accept`
- Source: entries/2026/05/29/bloom-filter-bloom_filter-remove.md

### [ACCEPT] cbf-remove-can-introduce-false-negatives
Unlike standard Bloom filters which never have false negatives, `CountingBloomFilter.remove()` can cause false negatives when two items share a hash position — removing one decrements the shared counter, potentially making the other appear absent
- Source: entries/2026/05/29/bloom-filter-bloom_filter-remove.md

### [REJECT] bloom-filter-double-hashing
Duplicate of existing `bloom-double-hashing`
- Source: entries/2026/05/29/bloom-filter-bloom_filter.md

### [REJECT] bloom-filter-counting-saturation-leak
Duplicate of existing `cbf-saturated-counters-are-permanent`
- Source: entries/2026/05/29/bloom-filter-bloom_filter.md

### [REJECT] bloom-filter-scalable-fpr-geometric
Duplicate of existing `scalable-bloom-tightens-fpr` which already captures the per-slice geometric FPR tightening; the convergence formula `p / (1 - ratio)` is a mathematical consequence
- Source: entries/2026/05/29/bloom-filter-bloom_filter.md

### [ACCEPT] bloom-filter-union-count-overestimates
`BloomFilter.union()` sets `_count = self._count + other._count`, which overstates distinct items when filters share elements; `estimate_count()` from bit density is more accurate for unioned filters
- Source: entries/2026/05/29/bloom-filter-bloom_filter.md

### [REJECT] bloom-filter-remove-is-two-pass
Duplicate of `cbf-remove-two-pass-atomicity` (proposed above)
- Source: entries/2026/05/29/bloom-filter-bloom_filter.md

### [ACCEPT] bloom-filter-no-false-negatives
`BloomFilter.__contains__` never returns False for an item that was previously `add()`-ed — the fundamental Bloom filter invariant, tested at scale with 1000 items
- Source: entries/2026/05/29/bloom-filter-test_bloom_filter.md

### [ACCEPT] bloom-filter-len-counts-insertions
`BloomFilter.__len__` counts total `add()` calls, not distinct items — adding the same item twice yields `len() == 2`, unlike `CountingBloomFilter` which has the same property tracked separately
- Source: entries/2026/05/29/bloom-filter-test_bloom_filter.md

### [ACCEPT] counting-bloom-filter-reference-counted
`CountingBloomFilter` uses reference-counting semantics: an item added N times requires N `remove()` calls before it tests as absent via `__contains__`
- Source: entries/2026/05/29/bloom-filter-test_bloom_filter.md

### [ACCEPT] bloom-filter-optimal-sizing-textbook
`BloomFilter` computes `bit_count` as `ceil(-n * ln(p) / ln(2)^2)` and `hash_count` as `round((m/n) * ln(2))`, matching the standard optimal formulas from Bloom filter literature
- Source: entries/2026/05/29/bloom-filter-test_bloom_filter.md

### [ACCEPT] bloom-filter-deterministic-hashing
The Bloom filter hash function is deterministic (not seeded with randomness), so two filters with identical parameters produce identical bit arrays for identical inputs — enabling equality comparison and reproducible serialization
- Source: entries/2026/05/29/bloom-filter-test_bloom_filter.md

### [ACCEPT] apply-byzantine-is-outbound-only
`_apply_byzantine` is applied exclusively to outgoing messages; incoming messages are never filtered through it, so a Byzantine node's internal state (prepared/committed sets) remains consistent even while it sends corrupted messages
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_apply_byzantine.md

### [ACCEPT] equivocating-mode-expands-message-count
In `EQUIVOCATING` mode, each broadcast message is replaced by N-1 targeted messages with per-peer distinct digests computed via `compute_digest`, meaning the output list can be larger than the input
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_apply_byzantine.md

### [ACCEPT] silent-mode-drops-all-outbound
`SILENT` Byzantine mode returns an empty list for all outbound messages, but the node still processes incoming messages and updates internal state — simulating a crash fault rather than a protocol violation
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_apply_byzantine.md

### [ACCEPT] wrong-digest-is-node-specific
`WRONG_DIGEST` mode produces `"bad_digest_{node_id}"`, so two Byzantine nodes with this mode produce different invalid digests rather than accidentally colluding on the same forged value
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_apply_byzantine.md

### [ACCEPT] byzantine-mode-is-immutable-after-construction
The `byzantine_mode` field is set in `__init__` and never modified during protocol execution, so a node's fault behavior cannot change mid-protocol — simplifying reasoning about safety guarantees
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_apply_byzantine.md


### [ACCEPT] pbft-prepared-threshold-is-2f
`_check_prepared` requires exactly 2f matching PREPAREs (not 2f+1) because the primary's PRE-PREPARE counts as implicit agreement toward the quorum of 2f+1.
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_check_prepared.md

### [ACCEPT] pbft-prepared-at-most-once
A `(view, seq)` pair can transition to "prepared" at most once per node, enforced by the `prepared_requests` set guard at the top of `_check_prepared`.
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_check_prepared.md

### [REJECT] pbft-prepare-digest-match-required
Duplicates existing belief `wrong-digest-cannot-reach-quorum` which already captures that mismatched digests are silently dropped and cannot contribute to quorum.
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_check_prepared.md

### [ACCEPT] pbft-commit-self-logged-before-broadcast
`_check_prepared` logs the node's own COMMIT into `message_log` and `phase_senders` before returning it for broadcast, so duplicate detection correctly rejects re-delivery of the node's own COMMIT.
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_check_prepared.md

### [ACCEPT] pbft-prepared-chains-to-committed
`_check_prepared` always calls `_check_committed` after marking prepared, enabling immediate commit if enough COMMIT messages arrived out of order before the node reached the prepared state.
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_check_prepared.md

### [ACCEPT] pbft-primary-skips-prepare
The primary sends PRE-PREPARE but not PREPARE; `_handle_prepare` rejects PREPAREs from the primary, and the primary's code path in `_handle_pre_prepare` skips sending a PREPARE — this asymmetry is why the prepared threshold is 2f not 2f+1.
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_check_prepared.md

### [ACCEPT] view-change-requires-2f-plus-1
The new primary only acts on a view change after collecting at least 2f+1 VIEW_CHANGE messages, matching the standard PBFT quorum requirement for liveness.
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_handle_view_change.md

### [ACCEPT] view-change-contagion
Receiving a VIEW_CHANGE from another node causes a non-primary node to broadcast its own VIEW_CHANGE if it hasn't already, propagating the vote through the cluster without requiring all nodes to independently detect the faulty primary.
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_handle_view_change.md

### [ACCEPT] view-never-decreases
`_handle_view_change` silently drops any message with `msg.view <= self.current_view`, enforcing monotonically non-decreasing view progression across all nodes.
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_handle_view_change.md

### [ACCEPT] reprepare-merges-by-sequence
When the new primary merges prepared sets from VIEW_CHANGE messages, it deduplicates by sequence number using first-writer-wins without verifying that all nodes prepared the same request at that sequence — relying on the PBFT safety proof that at most one request can be prepared per (view, seq).
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_handle_view_change.md

### [ACCEPT] only-new-primary-advances-view-in-vc
Non-primary nodes do not update `current_view` in `_handle_view_change`; they wait for the `NEW_VIEW` message to adopt the new view, leaving them in a liminal state that rejects normal-phase messages for both old and new views.
- Source: entries/2026/05/29/byzantine-fault-tolerance-pbft-_handle_view_change.md

### [ACCEPT] pbft-cluster-validates-n-3f-plus-1
`PBFTCluster` raises `ValueError` if `n` is not exactly `3f+1` or if more than `f` byzantine nodes are specified, enforcing the minimum redundancy required for Byzantine fault tolerance.
- Source: entries/2026/05/29/byzantine-fault-tolerance-test_pbft.md

### [ACCEPT] pbft-duplicate-messages-silently-dropped
`PBFTNode.receive_message` returns an empty list for duplicate or invalid messages (out-of-range sender IDs, already-seen tuples) rather than raising exceptions, consistent with Byzantine protocol design where you can't trust the sender.
- Source: entries/2026/05/29/byzantine-fault-tolerance-test_pbft.md

### [ACCEPT] pbft-executed-log-sequence-numbers-contiguous
Honest nodes execute requests with contiguous sequence numbers starting from 1, enforced by the protocol's ordering guarantees and verified by test assertions on the `_executed_log`.
- Source: entries/2026/05/29/byzantine-fault-tolerance-test_pbft.md

### [ACCEPT] pbft-view-change-preserves-prepared-requests
Requests that reach the prepared state survive view changes: the new primary collects prepared-but-uncommitted requests from VIEW_CHANGE messages and re-proposes them with fresh PRE_PREPARE messages in the new view.
- Source: entries/2026/05/29/byzantine-fault-tolerance-test_pbft.md

### [ACCEPT] pbft-submit-request-is-synchronous-simulation
`PBFTCluster.submit_request` runs the full three-phase protocol to completion deterministically in a single call (no real networking or async), returning a boolean success indicator.
- Source: entries/2026/05/29/byzantine-fault-tolerance-test_pbft.md

### [ACCEPT] cdc-log-sequence-numbers-survive-compaction
`CDCLog.compact()` preserves the original sequence numbers of surviving events; it does not renumber them, so consumer positions remain valid references after compaction.
- Source: entries/2026/05/29/change-data-capture-cdc.md

### [ACCEPT] cdc-consumer-poll-is-at-least-once
If a CDC consumer crashes mid-`poll()`, the position is not advanced, so all events from that batch will be reprocessed on the next call — providing at-least-once delivery with no deduplication.
- Source: entries/2026/05/29/change-data-capture-cdc.md

### [ACCEPT] cdc-snapshot-uses-sentinel-sequence-number
`create_snapshot` assigns `sequence_number=-1` to all synthetic INSERT events, distinguishing bootstrapped state from real log entries and enabling new consumers to load current state then seek to the live tail.
- Source: entries/2026/05/29/change-data-capture-cdc.md

### [ACCEPT] cdc-materialized-view-transform-none-deletes
When a `MaterializedView`'s transform function returns `None` for an INSERT or UPDATE event, the row is removed from the view rather than stored, acting as a filter on the derived table.
- Source: entries/2026/05/29/change-data-capture-cdc.md

### [ACCEPT] cdc-search-index-full-reindex-on-update
`SearchIndex` removes all old tokens and re-adds all new tokens on every UPDATE event, even if only non-indexed columns changed — correct but not optimized for partial changes.
- Source: entries/2026/05/29/change-data-capture-cdc.md

### [ACCEPT] cdc-before-after-contract
Every `ChangeEvent` follows a strict before/after convention: INSERT has `before=None`, DELETE has `after=None`, UPDATE has both populated as full row copies, making events self-contained without querying the source database.
- Source: entries/2026/05/29/change-data-capture-cdc.md

### [ACCEPT] cdc-consumer-position-is-next-sequence
A CDC consumer's `_position` tracks the *next* sequence number to read (not the last one read); after processing, it advances to `last_event.sequence_number + 1`.
- Source: entries/2026/05/29/change-data-capture-cdc.md

### [REJECT] crdt-tests-cover-20-scenarios
Test count is an unstable implementation detail that changes as tests are added or removed; not a durable architectural claim.
- Source: entries/2026/05/29/conflict-free-replicated-data-types-test_crdts.md

### [REJECT] lww-tiebreak-by-replica-id
Duplicates existing belief `lww-tiebreak-is-deterministic` which already captures that `LWWRegister.merge` uses `(timestamp, writer_id)` tuple comparison for total ordering.
- Source: entries/2026/05/29/conflict-free-replicated-data-types-test_crdts.md

### [ACCEPT] pncounter-composed-of-two-gcounters
PNCounter state is structured as two GCounter sub-counters (`p` for increments, `n` for decrements) with `value() = p - n`, confirming the standard decomposition from CRDT literature.
- Source: entries/2026/05/29/conflict-free-replicated-data-types-test_crdts.md

### [ACCEPT] crdt-merge-returns-self
`merge()` mutates and returns `self` (enabling chaining like `deepcopy(a).merge(b)`) rather than returning a new instance, which means callers must deepcopy before merging if they need to preserve the original state.
- Source: entries/2026/05/29/conflict-free-replicated-data-types-test_crdts.md

### [ACCEPT] lww-auto-increments-timestamp
Sequential `LWWRegister.set()` calls without explicit timestamps produce monotonically increasing timestamps (auto-incremented), so ordering is preserved even without caller-supplied clocks.
- Source: entries/2026/05/29/conflict-free-replicated-data-types-test_crdts.md

### [ACCEPT] orset-remove-returns-false-for-absent
`ORSet.remove()` returns `False` for a non-existent element rather than raising an exception, treating removal of absent elements as a no-op with a boolean status indicator.
- Source: entries/2026/05/29/conflict-free-replicated-data-types-test_crdts.md

### [ACCEPT] crdt-eq-compares-semantic-state
All four CRDT types implement `__eq__` to compare semantic state (not object identity), enabling `CRDTReplicaGroup.all_converged()` and the semilattice property tests to verify convergence via equality checks.
- Source: entries/2026/05/29/conflict-free-replicated-data-types-test_crdts.md


### [ACCEPT] consistent-hash-ring-sorted-invariant
`_ring_positions` is maintained in sorted order at all times via `bisect` insertion; `_ring_nodes` is kept parallel to it with `len(_ring_positions) == len(_ring_nodes)` always holding.
- Source: entries/2026/05/29/consistent-hashing-consistent_hashing-ConsistentHashRing.md

### [ACCEPT] consistent-hash-ring-lookup-is-olog-v
Key lookup via `get_node` is O(log V) where V is total virtual nodes, using `bisect` on the sorted position list; node addition is O(V) per vnode due to `list.insert`.
- Source: entries/2026/05/29/consistent-hashing-consistent_hashing-ConsistentHashRing.md

### [REJECT] consistent-hash-ring-vnode-count-scaled-by-weight
Duplicates existing belief `ch-vnode-count-is-weighted`.
- Source: entries/2026/05/29/consistent-hashing-consistent_hashing-ConsistentHashRing.md

### [REJECT] consistent-hash-ring-get-nodes-skips-same-physical
Duplicates existing belief `ch-preference-list-skips-duplicates`.
- Source: entries/2026/05/29/consistent-hashing-consistent_hashing-ConsistentHashRing.md

### [REJECT] consistent-hash-ring-transfer-map-approximate
Duplicates existing belief `ch-transfer-maps-are-descriptive`.
- Source: entries/2026/05/29/consistent-hashing-consistent_hashing-ConsistentHashRing.md

### [ACCEPT] consistent-hash-ring-not-thread-safe
`ConsistentHashRing` has no synchronization; concurrent `add_node`/`remove_node` calls corrupt the sorted `_ring_positions` and `_ring_nodes` lists.
- Source: entries/2026/05/29/consistent-hashing-consistent_hashing-ConsistentHashRing.md

### [ACCEPT] consistent-hash-ring-no-positive-validation
`ConsistentHashRing` does not validate that `num_vnodes`, `weight`, or `replication_factor` are positive; zero or negative values silently produce degenerate ring states (e.g., a node with zero vnodes exists in `_nodes` but owns no ring arc).
- Source: entries/2026/05/29/consistent-hashing-consistent_hashing-ConsistentHashRing.md

### [ACCEPT] consistent-hash-ring-uses-md5-truncated-to-32-bit
The ring hashes keys and vnode identifiers using `hashlib.md5` truncated to 32 bits via `& 0xFFFFFFFF`, chosen for distribution quality rather than security.
- Source: entries/2026/05/29/consistent-hashing-consistent_hashing-ConsistentHashRing.md

### [ACCEPT] consistent-hash-ring-uses-32-bit-space
The ring's hash space is `[0, 2^32)`, as validated by `test_ring_position_valid_range`.
- Source: entries/2026/05/29/consistent-hashing-test_consistent_hashing.md

### [ACCEPT] consistent-hash-get-nodes-first-equals-get-node
`get_nodes(key)[0]` always equals `get_node(key)` — the preference list's head is the primary replica.
- Source: entries/2026/05/29/consistent-hashing-test_consistent_hashing.md

### [ACCEPT] consistent-hash-add-node-is-idempotent
Adding a node that already exists silently returns an empty transfer map and does not change the ring's node count, vnode positions, or key assignments.
- Source: entries/2026/05/29/consistent-hashing-test_consistent_hashing.md

### [ACCEPT] consistent-hash-replication-requires-sufficient-physical-nodes
`get_nodes()` raises `ValueError` rather than silently under-replicating when `replication_factor` exceeds the number of physical nodes on the ring.
- Source: entries/2026/05/29/consistent-hashing-test_consistent_hashing.md

### [REJECT] consistent-hash-membership-change-returns-transfer-map
Duplicates existing belief `ch-transfer-maps-are-descriptive`.
- Source: entries/2026/05/29/consistent-hashing-test_consistent_hashing.md

### [ACCEPT] consistent-hash-minimal-redistribution
Adding an Nth node to the ring moves approximately 1/N of keys, not a full reshuffle — the defining property that makes consistent hashing useful for dynamic cluster membership.
- Source: entries/2026/05/29/consistent-hashing-test_consistent_hashing.md

### [REJECT] live-projection-synchronous-dispatch
Duplicates existing beliefs `live-projection-is-synchronous` and `subscriber-dispatch-is-synchronous`.
- Source: entries/2026/05/29/event-sourcing-store-event_store-LiveProjection.md

### [ACCEPT] live-projection-no-unsubscribe
Once constructed, a `LiveProjection` cannot be unsubscribed from its store; the callback reference persists in `_subscribers` indefinitely with no removal mechanism.
- Source: entries/2026/05/29/event-sourcing-store-event_store-LiveProjection.md

### [ACCEPT] live-projection-idempotent-by-position
`LiveProjection._on_event` guards against double-processing via `event_id <= _position`, making it safe to overlap `catch_up()` and live subscription without duplicate handler invocations.
- Source: entries/2026/05/29/event-sourcing-store-event_store-LiveProjection.md

### [ACCEPT] live-projection-position-advances-without-handler
Events with no registered handler still advance `LiveProjection._position`, preventing re-evaluation on subsequent `catch_up()` calls — position tracks last *seen* event, not last *handled* event.
- Source: entries/2026/05/29/event-sourcing-store-event_store-LiveProjection.md

### [ACCEPT] live-projection-no-error-boundary
Handler exceptions in `LiveProjection._on_event` propagate through `EventStore.append()` to the caller; the projection's `_position` is not updated, leaving it behind but recoverable via `catch_up()`.
- Source: entries/2026/05/29/event-sourcing-store-event_store-LiveProjection.md

### [ACCEPT] persist-event-no-fsync
`_persist_event` writes to the OS buffer via `file.write()` but never calls `fsync`; event data can be lost on crash between write and OS flush to disk.
- Source: entries/2026/05/29/event-sourcing-store-event_store-_persist_event.md

### [ACCEPT] persist-event-per-call-open
The persist file is opened in append mode and closed on every single `_persist_event` call, not held open across the store's lifetime — safe but incurs per-event open/close overhead.
- Source: entries/2026/05/29/event-sourcing-store-event_store-_persist_event.md

### [ACCEPT] persist-before-notify
Disk persistence via `_persist_event` happens before subscriber notification: an event is written to the NDJSON file before any `LiveProjection` or other subscriber sees it.
- Source: entries/2026/05/29/event-sourcing-store-event_store-_persist_event.md

### [ACCEPT] persist-after-memory
The event is appended to `self._events` (in-memory) before `_persist_event` writes it to disk; a disk write failure leaves the in-memory store ahead of the on-disk log with no rollback.
- Source: entries/2026/05/29/event-sourcing-store-event_store-_persist_event.md

### [REJECT] batch-persist-non-atomic
Duplicates existing belief `batch-append-not-crash-safe`.
- Source: entries/2026/05/29/event-sourcing-store-event_store-_persist_event.md

### [REJECT] append-batch-defers-subscribers
Duplicates existing belief `batch-notifies-after-all-stored`.
- Source: entries/2026/05/29/event-sourcing-store-event_store-append_batch.md

### [REJECT] append-batch-not-transactional
Duplicates existing belief `batch-append-not-crash-safe`.
- Source: entries/2026/05/29/event-sourcing-store-event_store-append_batch.md

### [REJECT] event-id-is-global-sequence
Duplicates existing belief `event-ids-are-1-based-sequential`.
- Source: entries/2026/05/29/event-sourcing-store-event_store-append_batch.md

### [ACCEPT] append-batch-shares-references
Returned `Event` objects from `append_batch` are the same instances stored in `_events`; no defensive copy is made for either the event or its `data` dict, so caller mutation corrupts store state.
- Source: entries/2026/05/29/event-sourcing-store-event_store-append_batch.md

### [ACCEPT] concurrency-check-is-pre-mutation
The `expected_version` optimistic concurrency check in `append_batch` runs before any state mutation, so a `ConcurrencyConflict` exception is safe and leaves the store completely unchanged.
- Source: entries/2026/05/29/event-sourcing-store-event_store-append_batch.md

### [ACCEPT] event-store-single-threaded-assumption
`EventStore` assumes single-threaded access: `event_id = len(self._events) + 1` is a race condition under concurrency, and no locking protects `_events`, `_streams`, or subscriber notification.
- Source: entries/2026/05/29/event-sourcing-store-event_store-append_batch.md


### [ACCEPT] projection-catch-up-advances-position-for-unhandled-events
`catch_up` advances `_position` for every event, not just those with registered handlers, so unhandled event types don't create replay gaps
- Source: entries/2026/05/29/event-sourcing-store-event_store-catch_up.md

### [ACCEPT] projection-catch-up-poison-event-stalls
If a handler raises during `catch_up`, `_position` is not advanced past the failing event, causing all subsequent `catch_up` calls to re-encounter and re-fail on the same event indefinitely
- Source: entries/2026/05/29/event-sourcing-store-event_store-catch_up.md

### [ACCEPT] projection-snapshot-interval-counts-all-events
The snapshot interval counter increments for every event processed during `catch_up`, including those without handlers, so snapshot frequency is based on total event throughput not just handled events
- Source: entries/2026/05/29/event-sourcing-store-event_store-catch_up.md

### [ACCEPT] projection-catch-up-assumes-sequential-ids
`catch_up` uses `from_position=self._position + 1` arithmetic that assumes `event_id` values are contiguous integers with no gaps; IDs with gaps would cause events to be silently skipped
- Source: entries/2026/05/29/event-sourcing-store-event_store-catch_up.md

### [ACCEPT] fencing-token-counter-is-global
The fencing token counter is global across all locks, not per-lock; tokens issued for different locks are comparable and strictly ordered by acquisition time
- Source: entries/2026/05/29/fencing-tokens-test_fencing_tokens.md

### [REJECT] fenced-server-rejects-by-high-water-mark
Duplicates existing belief `fencing-rejects-stale-writes`
- Source: entries/2026/05/29/fencing-tokens-test_fencing_tokens.md

### [ACCEPT] fencing-token-tracking-is-per-resource
Token high-water marks on `FencedResourceServer` are scoped per resource identifier, so a lower token can succeed on a different resource than the one that saw a higher token
- Source: entries/2026/05/29/fencing-tokens-test_fencing_tokens.md

### [REJECT] lock-renewal-does-not-increment-counter
Duplicates existing belief `renew-preserves-token-number`
- Source: entries/2026/05/29/fencing-tokens-test_fencing_tokens.md

### [REJECT] fencing-errors-use-return-values-not-exceptions
Duplicates existing belief `fencing-no-exceptions-on-denial`
- Source: entries/2026/05/29/fencing-tokens-test_fencing_tokens.md

### [ACCEPT] receive-gossip-death-is-sticky
Once a node's local status is `dead`, `receive_gossip` will never change it back to `alive`, even if the remote has a higher heartbeat counter — preventing flapping after failure declaration
- Source: entries/2026/05/29/gossip-protocol-gossip_protocol-receive_gossip.md

### [REJECT] receive-gossip-skips-unknown-dead-nodes
Duplicates existing belief `gossip-dead-nodes-filtered-on-receive`
- Source: entries/2026/05/29/gossip-protocol-gossip_protocol-receive_gossip.md

### [REJECT] receive-gossip-equal-counter-death-acceptance
Duplicates existing belief `gossip-merge-uses-heartbeat-monotonicity` which already captures the equal-counter death exception
- Source: entries/2026/05/29/gossip-protocol-gossip_protocol-receive_gossip.md

### [ACCEPT] receive-gossip-deep-copies-on-insert
`receive_gossip` deep-copies remote entries before inserting them into local membership, extending the isolation guarantee beyond the `send_gossip`/`join`/`get_membership_list` paths covered by `gossip-deep-copy-isolation`
- Source: entries/2026/05/29/gossip-protocol-gossip_protocol-receive_gossip.md

### [ACCEPT] receive-gossip-no-self-exclusion
Unlike `detect_failures` which skips the node's own ID, `receive_gossip` has no self-exclusion guard — if the remote gossip contains an entry for the receiver, it is merged normally
- Source: entries/2026/05/29/gossip-protocol-gossip_protocol-receive_gossip.md

### [REJECT] gossip-failure-detection-is-three-phase
Duplicates existing belief `gossip-node-status-lifecycle`
- Source: entries/2026/05/29/gossip-protocol-test_gossip_protocol.md

### [REJECT] gossip-heartbeat-wins-resolve-conflicts
Duplicates existing belief `gossip-merge-uses-heartbeat-monotonicity`
- Source: entries/2026/05/29/gossip-protocol-test_gossip_protocol.md

### [ACCEPT] gossip-convergence-is-olog-n
Full membership convergence occurs within O(log N) gossip rounds, empirically bounded by `5 * log₂(N) + 5` rounds in the test suite
- Source: entries/2026/05/29/gossip-protocol-test_gossip_protocol.md

### [ACCEPT] gossip-cluster-uses-logical-time
The gossip simulation uses explicit logical timestamps passed as parameters (not wall-clock time), with a fixed RNG seed for deterministic gossip partner selection, following the same simulated-time pattern as hinted-handoff
- Source: entries/2026/05/29/gossip-protocol-test_gossip_protocol.md

### [ACCEPT] gossip-cleanup-removes-entry-entirely
After `t_cleanup` elapses, a dead node's record is fully deleted from membership (not just flagged), which enables clean rejoin with a fresh identity
- Source: entries/2026/05/29/gossip-protocol-test_gossip_protocol.md

### [ACCEPT] bitcask-rotation-is-soft-cap
File rotation is checked before each write, so the active file can exceed `max_file_size` by up to one record's worth of bytes; the limit is a soft cap, not a hard one
- Source: entries/2026/05/29/hash-index-storage-bitcask-_maybe_rotate.md

### [ACCEPT] bitcask-rotation-preserves-read-handles
When the active file is rotated, its read handle in `file_handles` is preserved so existing `KeyEntry` references pointing to the old file remain valid
- Source: entries/2026/05/29/hash-index-storage-bitcask-_maybe_rotate.md

### [ACCEPT] bitcask-sequential-file-ids
File IDs are assigned by incrementing `active_file_id` by 1 with no gap detection or collision check; compaction updates `active_file_id` to avoid collisions after renumbering
- Source: entries/2026/05/29/hash-index-storage-bitcask-_maybe_rotate.md


### [ACCEPT] bitcask-record-format-symmetry
`_read_record` and `_write_record` share the same binary layout (`<dII` header + key bytes + value bytes); a change to either must be mirrored in the other.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_read_record.md

### [ACCEPT] bitcask-reader-not-threadsafe
File handles in `self.file_handles` are shared and mutable (via `seek`), so concurrent calls to `_read_record` on the same `file_id` would race.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_read_record.md

### [REJECT] bitcask-read-single-io
Duplicate of existing `bitcask-get-single-disk-seek` which already captures the O(1) I/O property.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_read_record.md

### [REJECT] bitcask-tombstone-empty-string
Already exists as `bitcask-tombstone-is-empty-string`.
- Source: entries/2026/05/29/hash-index-storage-bitcask-_read_record.md

### [ACCEPT] hinted-handoff-sloppy-quorum-writes-to-non-preferred
When sloppy quorum is enabled and preferred replicas are down, writes succeed by routing to non-preferred substitute nodes that store hints for later delivery.
- Source: entries/2026/05/29/hinted-handoff-test_hinted_handoff.md

### [ACCEPT] hinted-handoff-hint-ttl-exclusive-boundary
Hint expiration uses exclusive comparison: a hint created at time `t` with TTL `d` survives at `t + d - 1` but is expired at `t + d`.
- Source: entries/2026/05/29/hinted-handoff-test_hinted_handoff.md

### [ACCEPT] hinted-handoff-strict-quorum-rejects-insufficient-preferred
With `sloppy_quorum=False`, a write fails if fewer than `write_quorum` preferred nodes are available, even if non-preferred nodes could serve.
- Source: entries/2026/05/29/hinted-handoff-test_hinted_handoff.md

### [ACCEPT] hinted-handoff-hint-loss-is-silent-data-loss
If a hint node crashes and its hints are lost before handoff, the target node never receives the data and no error is raised — requiring anti-entropy repair (e.g., Merkle trees) to detect and fix.
- Source: entries/2026/05/29/hinted-handoff-test_hinted_handoff.md

### [REJECT] hinted-handoff-hints-cleared-after-delivery
Duplicate of existing `hinted-handoff-hint-cleanup-all-or-nothing` which already covers post-handoff hint removal.
- Source: entries/2026/05/29/hinted-handoff-test_hinted_handoff.md

### [ACCEPT] reaches-uses-identity-not-equality
`_reaches` uses `is` (object identity) to find the source event, not `==`, because dataclass equality would conflate distinct events with identical field values.
- Source: entries/2026/05/29/lamport-clocks-lamport-_reaches.md

### [ACCEPT] reaches-walks-two-edge-types
The causal graph has two edge types: `_parent` (same-node sequential) and `_cause` (cross-node send→receive), and `_reaches` must follow both to correctly determine happens-before.
- Source: entries/2026/05/29/lamport-clocks-lamport-_reaches.md

### [ACCEPT] happens-before-requires-two-reaches-calls
`happens_before` calls `_reaches` in both directions to distinguish "a before b", "b before a", and "concurrent" — a single call can only confirm or deny one direction.
- Source: entries/2026/05/29/lamport-clocks-lamport-_reaches.md

### [ACCEPT] event-graph-acyclicity-by-construction
`_reaches` assumes the event graph is a DAG with no cycle detection; this is safe because `Node` methods only ever link to previously-created events via `_parent` and `_cause`.
- Source: entries/2026/05/29/lamport-clocks-lamport-_reaches.md

### [ACCEPT] lamport-receive-tick-max-merge
`LamportClock.receive_tick(remote_ts)` computes `max(local, remote_ts) + 1`, ensuring the clock advances past both the local and remote state on every receive.
- Source: entries/2026/05/29/lamport-clocks-test_lamport.md

### [ACCEPT] lamport-send-creates-receive
`Node.send_message(target, payload)` has a side effect on the target node: it appends a RECEIVE event to the target's event log and advances the target's clock.
- Source: entries/2026/05/29/lamport-clocks-test_lamport.md

### [ACCEPT] lamport-happens-before-returns-tristate
`happens_before(a, b, events)` returns `True` (a causes b), `False` (b causes a), or `None` (concurrent) — modeling a strict partial order with explicit concurrency detection.
- Source: entries/2026/05/29/lamport-clocks-test_lamport.md

### [ACCEPT] lamport-total-order-tiebreak-by-node-id
`total_order()` sorts events by `(timestamp, node_id)`, using the node identifier as a deterministic tiebreaker for concurrent events with equal timestamps.
- Source: entries/2026/05/29/lamport-clocks-test_lamport.md

### [ACCEPT] lamport-mutex-lowest-timestamp-priority
`LamportMutex` grants critical section access to the requesting node with the lowest timestamp, with other requesters waiting until the holder releases.
- Source: entries/2026/05/29/lamport-clocks-test_lamport.md

### [REJECT] split-brain-highest-id-wins
Duplicate of existing `bully-highest-id-wins`.
- Source: entries/2026/05/29/leader-election-leader_election-_resolve_split_brain.md

### [ACCEPT] split-brain-runs-after-message-drain
`_resolve_split_brain` executes only after the tick's `while all_messages` delivery loop has fully drained, never mid-delivery.
- Source: entries/2026/05/29/leader-election-leader_election-_resolve_split_brain.md

### [ACCEPT] split-brain-delivers-two-hops
Election messages from demoted leaders are delivered one level deep (election message + immediate response), not recursively to convergence; the next `tick()` finishes any remaining propagation.
- Source: entries/2026/05/29/leader-election-leader_election-_resolve_split_brain.md

### [ACCEPT] split-brain-not-recorded-in-history
Elections triggered by `_resolve_split_brain` do not call `_record_leader`, so leadership changes from split-brain resolution are absent from `election_history`.
- Source: entries/2026/05/29/leader-election-leader_election-_resolve_split_brain.md

### [ACCEPT] split-brain-ignores-unavailable-nodes
Only nodes where `is_available()` is `True` are considered when detecting and resolving split-brain; crashed nodes' stale leader state is ignored.
- Source: entries/2026/05/29/leader-election-leader_election-_resolve_split_brain.md


### [REJECT] bully-election-highest-id-wins
Duplicates existing belief `bully-highest-id-wins`
- Source: entries/2026/05/29/leader-election-test_leader_election.md

### [ACCEPT] bully-tests-use-deterministic-simulation
Leader election tests control time via an explicit `start_time` parameter to `run_until_leader()` rather than real timers or sleeps, making the entire test suite deterministic with no flaky timing
- Source: entries/2026/05/29/leader-election-test_leader_election.md

### [ACCEPT] bully-election-terms-monotonic
Election terms in `get_election_history()` are enforced to be monotonically non-decreasing across successive elections, validated by `test_terms_increase_monotonically`
- Source: entries/2026/05/29/leader-election-test_leader_election.md

### [REJECT] bully-test-imports-unused-symbols
Trivial import detail, not an architectural invariant that would break things if violated
- Source: entries/2026/05/29/leader-election-test_leader_election.md

### [ACCEPT] anti-entropy-skips-unavailable-nodes
Anti-entropy repair only reads from and writes to nodes where `is_available` is true; down nodes are excluded from both key discovery and repair propagation
- Source: entries/2026/05/29/leaderless-replication-dynamo-anti_entropy_repair.md

### [ACCEPT] anti-entropy-does-not-advance-versions
`anti_entropy_repair()` propagates existing `VersionedValue` entries without modifying `_version_counters`, so it never creates new versions — only copies the highest existing version to lagging replicas
- Source: entries/2026/05/29/leaderless-replication-dynamo-anti_entropy_repair.md

### [ACCEPT] anti-entropy-counts-per-node-writes
The repair count returned by `anti_entropy_repair()` reflects individual node writes, not unique keys repaired; one stale key across N nodes counts as N repairs
- Source: entries/2026/05/29/leaderless-replication-dynamo-anti_entropy_repair.md

### [REJECT] anti-entropy-complements-read-repair
Design rationale describing the relationship between two mechanisms, not a testable code invariant
- Source: entries/2026/05/29/leaderless-replication-dynamo-anti_entropy_repair.md

### [ACCEPT] anti-entropy-highest-version-wins
Anti-entropy repair assumes a total ordering on versions where the highest version number is authoritative, which is safe because `put()` uses a global per-key version counter ensuring no two writes produce the same version
- Source: entries/2026/05/29/leaderless-replication-dynamo-anti_entropy_repair.md

### [ACCEPT] dynamo-write-fans-to-all
`DynamoCluster.put()` sends writes to every available node, not just W of them; the quorum check gates on acknowledgment count, not send count, maximizing replica consistency
- Source: entries/2026/05/29/leaderless-replication-dynamo.md

### [ACCEPT] dynamo-version-counter-is-global
Version assignment uses a single coordinator-level counter per key (`_version_counters`), not per-replica counters, which prevents version conflicts but requires a centralized coordinator
- Source: entries/2026/05/29/leaderless-replication-dynamo.md

### [ACCEPT] dynamo-read-repair-is-eager
`DynamoCluster.get()` repairs all stale replicas reachable during the read, not just enough to satisfy the quorum — every replica with a version below the max is updated
- Source: entries/2026/05/29/leaderless-replication-dynamo.md

### [ACCEPT] dynamo-failed-write-rolls-back-version
When `put()` fails to meet write quorum, it decrements the version counter to prevent version gaps, then raises `QuorumNotMet`
- Source: entries/2026/05/29/leaderless-replication-dynamo.md

### [ACCEPT] dynamo-hints-single-homed
Hinted handoffs for all unavailable nodes are stored on `available_nodes[0]` only, creating a single point of responsibility for hint delivery and a potential hotspot
- Source: entries/2026/05/29/leaderless-replication-dynamo.md

### [REJECT] dynamo-version-rollback-on-failed-write
Duplicates `dynamo-failed-write-rolls-back-version` accepted from the dynamo.py file entry
- Source: entries/2026/05/29/leaderless-replication-test_dynamo_tester.md

### [ACCEPT] dynamo-conflict-detection-same-version-different-values
When multiple replicas hold different values at the same max version number, `ReadResult.is_conflict` is `True` and `value` is a list of all conflicting values rather than a single resolved value
- Source: entries/2026/05/29/leaderless-replication-test_dynamo_tester.md

### [ACCEPT] dynamo-sloppy-quorum-opt-in
Hinted handoff only occurs when `DynamoCluster` is constructed with `sloppy_quorum=True`; without it, `deliver_hints()` returns 0 and offline nodes receive no data until anti-entropy runs
- Source: entries/2026/05/29/leaderless-replication-test_dynamo_tester.md

### [ACCEPT] dynamo-per-key-versioning
Version counters in `DynamoCluster._version_counters` are scoped per key, not global across the cluster; independent keys maintain independent version sequences
- Source: entries/2026/05/29/leaderless-replication-test_dynamo_tester.md

### [REJECT] dynamo-read-repair-on-get
Overlaps with accepted `dynamo-read-repair-is-eager` which covers the same behavior with a stronger claim
- Source: entries/2026/05/29/leaderless-replication-test_dynamo_tester.md

### [ACCEPT] bitcask-segment-discovery-is-filesystem-based
`_find_existing_segments` uses `os.listdir` as the source of truth for which segments exist; there is no persistent manifest file — the filesystem is the manifest
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_find_existing_segments.md

### [REJECT] bitcask-segment-sort-order-is-by-id
Duplicates existing beliefs `bitcask-rebuild-sorts-ascending` and `bitcask-recovery-order-is-ascending` which already establish ascending ID ordering
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_find_existing_segments.md

### [ACCEPT] bitcask-segment-naming-must-sync
The filename parsing in `_find_existing_segments` (prefix/suffix slicing to extract segment ID) must stay in sync with the format string in `_segment_path`; no shared constant enforces this coupling
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_find_existing_segments.md

### [ACCEPT] bitcask-no-validation-on-segment-filenames
If a file matching `segment_*.dat` contains non-numeric characters in the ID position, `_find_existing_segments` crashes with `ValueError`; no validation guards against malformed filenames
- Source: entries/2026/05/29/log-structured-hash-table-bitcask-_find_existing_segments.md


### [ACCEPT] scan-range-sorted-order-assumption
`_scan_range_for_key` assumes entries between `start` and `end` are sorted by key ascending; it breaks early on `k > key`, so unsorted data causes silent false negatives
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-_scan_range_for_key.md

### [REJECT] scan-range-tombstone-transparent
Duplicates existing belief `sstable-get-does-not-filter-tombstones` which already captures that tombstone interpretation is the caller's responsibility
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-_scan_range_for_key.md

### [REJECT] scan-range-no-state-mutation
Trivial — a read method being read-only is expected behavior, not an invariant that would break things if violated
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-_scan_range_for_key.md

### [ACCEPT] scan-range-corrupt-data-silent
If `_read_entry` encounters truncated data mid-range, `_scan_range_for_key` silently returns `(False, None)` rather than raising an error, making corruption indistinguishable from a missing key
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-_scan_range_for_key.md

### [REJECT] sstable-get-does-not-filter-tombstones
Already exists in the accepted beliefs registry
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-get.md

### [REJECT] sparse-index-bisect-narrows-to-interval-range
Substantially covered by existing belief `sstable-lookup-is-two-phase` which describes the binary-search-then-linear-scan pattern with O(log(N/B) + B) cost
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-get.md

### [ACCEPT] sstable-get-requires-prior-load-index
For SSTables loaded from disk, `load_index()` must be called before `get()` or every lookup silently returns `(False, None)` — the method does not call `load_index()` itself
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-get.md

### [ACCEPT] sstable-get-rebuilds-key-list-every-call
`SSTable.get` extracts sparse index keys into a fresh list on every call rather than caching them, making each lookup O(m) in index size before the binary search begins
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-get.md

### [REJECT] sparse-index-sorted-order-invariant
Overlaps with `scan-range-sorted-order-assumption` and the general sorted-order invariants already captured across SSTable beliefs
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-get.md

### [REJECT] wal-replay-tolerates-partial-writes
Duplicates existing belief `lsm-wal-crash-tolerant-replay`
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-replay.md

### [REJECT] wal-no-checksum-verification
Duplicates existing beliefs `lsm-wal-has-no-checksums` and `lsm-wal-has-no-integrity-check`
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-replay.md

### [REJECT] wal-replay-order-matches-append-order
Sequential order preservation is inherent to reading a sequential log file; not an invariant that could be violated by a code change
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-replay.md

### [REJECT] wal-replay-is-read-only
Trivial — a replay/recovery method being read-only is expected design, not a breakable invariant
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-replay.md

### [ACCEPT] wal-flush-truncate-replay-safety
If a crash occurs between SSTable flush and WAL truncation, replay re-inserts already-persisted entries into the memtable, which is safe because memtable values shadow SSTable values during reads (last-writer-wins)
- Source: entries/2026/05/29/log-structured-merge-tree-lsm-replay.md

### [ACCEPT] broadcast-join-loads-small-side-at-construction
`BroadcastHashJoin` receives the small dataset at construction time and builds a hash table; the large dataset is streamed through `.join()`, reflecting the asymmetric API where the small side must be available upfront
- Source: entries/2026/05/29/map-side-join-test_map_side_joins.md

### [REJECT] all-three-join-strategies-produce-identical-inner-join-results
Duplicates existing belief `map-side-join-three-strategies` which already states all three produce identical inner-join results
- Source: entries/2026/05/29/map-side-join-test_map_side_joins.md

### [REJECT] missing-join-keys-are-skipped-not-errored
Duplicates existing belief `map-side-join-missing-key-skip`
- Source: entries/2026/05/29/map-side-join-test_map_side_joins.md

### [REJECT] field-name-conflicts-resolved-with-left-right-prefix
Duplicates existing belief `map-side-join-conflict-prefix`
- Source: entries/2026/05/29/map-side-join-test_map_side_joins.md

### [ACCEPT] sort-merge-join-detects-presorted-input
`SortMergeJoin` inspects whether input is already sorted and reports this via `stats["sorted_input"]`, allowing it to skip the sort step on pre-sorted data
- Source: entries/2026/05/29/map-side-join-test_map_side_joins.md

### [ACCEPT] mapreduce-results-deterministic
`MapReduceJob.run()` produces identical output regardless of `num_mappers`/`num_reducers` configuration for the same input data — the shuffle/partition step is deterministic
- Source: entries/2026/05/29/mapreduce-framework-test_mapreduce.md

### [ACCEPT] mapreduce-combiner-transparent
Applying a combiner never changes final results; it only reduces intermediate record volume — the combiner must be semantically invisible
- Source: entries/2026/05/29/mapreduce-framework-test_mapreduce.md

### [ACCEPT] mapreduce-strict-mode-default
`MapReduceJob` defaults to strict mode where mapper/reducer exceptions propagate to the caller; `fault_tolerant=True` must be explicitly set to enable silent error skipping
- Source: entries/2026/05/29/mapreduce-framework-test_mapreduce.md

### [ACCEPT] mapreduce-run-accepts-filepath
`MapReduceJob.run()` accepts either a list of `(key, value)` tuples or a file path string as input, with the file path handled transparently
- Source: entries/2026/05/29/mapreduce-framework-test_mapreduce.md

### [ACCEPT] mapreduce-stats-tracks-records
`job.stats` tracks `map_input_records`, `map_output_records`, `reduce_output_records`, worker counts, and elapsed time after each run
- Source: entries/2026/05/29/mapreduce-framework-test_mapreduce.md


### [ACCEPT] merkle-diff-filters-padding
`diff()` excludes padding indices beyond `max(self._leaf_count, other._leaf_count)`, using `max` (not `min`) so positions where one tree has data and the other has `EMPTY_HASH` padding are still reported as diffs
- Source: entries/2026/05/29/merkle-tree-merkle_tree-diff.md

### [ACCEPT] merkle-diff-is-pure
`diff()` and `_diff_recursive` are read-only — they mutate neither tree, allocate only a local result list, and produce no side effects
- Source: entries/2026/05/29/merkle-tree-merkle_tree-diff.md

### [ACCEPT] merkle-diff-order-is-ascending
Differing leaf indices returned by `diff()` are in ascending order as a consequence of left-to-right DFS traversal in `_diff_recursive`
- Source: entries/2026/05/29/merkle-tree-merkle_tree-diff.md

### [ACCEPT] merkle-diff-asymmetric-leaf-counts
Two trees with different `_leaf_count` but the same `_padded_size` can be diffed; positions where one tree has data and the other has `EMPTY_HASH` padding are reported as diffs
- Source: entries/2026/05/29/merkle-tree-merkle_tree-diff.md

### [ACCEPT] merkle-proof-sibling-order-bottom-up
`get_proof` returns siblings ordered from leaf level to root level (bottom-up), and `verify_proof` consumes them in that same order
- Source: entries/2026/05/29/merkle-tree-merkle_tree-get_proof.md

### [ACCEPT] merkle-proof-direction-is-sibling-position
The `direction` field in each proof sibling tuple indicates where the **sibling** sits (left or right), not the current node — this controls hash concatenation order during verification
- Source: entries/2026/05/29/merkle-tree-merkle_tree-get_proof.md

### [ACCEPT] merkle-proof-length-equals-height
A proof for a tree with `_padded_size = 2^h` contains exactly `h` sibling entries, one per tree level excluding the root
- Source: entries/2026/05/29/merkle-tree-merkle_tree-get_proof.md

### [ACCEPT] merkle-proof-is-snapshot
Merkle proofs are point-in-time snapshots; a proof generated before `update_leaf` will fail verification against the new root hash, and nothing invalidates the stale proof in-place
- Source: entries/2026/05/29/merkle-tree-merkle_tree-get_proof.md

### [ACCEPT] merkle-get-proof-rejects-padding-indices
`get_proof` bounds-checks against `_leaf_count` (real leaves), not `_padded_size`, so proofs cannot be generated for padding slots even though they have valid hashes in the backing array
- Source: entries/2026/05/29/merkle-tree-merkle_tree-get_proof.md

### [ACCEPT] merkle-tree-array-indexing
The tree uses 0-indexed implicit array layout where children of node `i` are at `2i+1` and `2i+2`, and leaves start at index `padded_size - 1`
- Source: entries/2026/05/29/merkle-tree-merkle_tree.md

### [ACCEPT] key-range-merkle-sorts-by-key
`KeyRangeMerkleTree` sorts input pairs by key before building, so two trees with identical key-value content always produce the same root hash regardless of insertion order
- Source: entries/2026/05/29/merkle-tree-merkle_tree.md

### [REJECT] merkle-internal-nodes-hash-hex-concatenation
Duplicate of existing `merkle-hashes-over-hex-not-bytes` which already captures that internal nodes hash concatenated hex digest strings rather than raw bytes
- Source: entries/2026/05/29/merkle-tree-merkle_tree.md

### [ACCEPT] merkle-tree-height-formula
A `MerkleTree` with 4 leaves has `height == 2`, indicating height counts internal levels (log₂ of padded leaf count)
- Source: entries/2026/05/29/merkle-tree-test_merkle.md

### [ACCEPT] merkle-diff-returns-leaf-indices
`MerkleTree.diff()` returns a list of integer leaf indices where the two trees diverge, not subtree nodes or hash pairs
- Source: entries/2026/05/29/merkle-tree-test_merkle.md

### [REJECT] merkle-proof-sibling-count-is-log-n
Duplicate of `merkle-proof-length-equals-height` — same claim expressed differently (log₂(n) siblings vs h sibling entries); the height-based framing is more precise for non-power-of-2 cases
- Source: entries/2026/05/29/merkle-tree-test_merkle.md

### [ACCEPT] merkle-tree-supports-non-power-of-two
The implementation handles non-power-of-2 leaf counts (tested with 1 and 3 leaves) via internal padding to the next power of 2 using `EMPTY_HASH`
- Source: entries/2026/05/29/merkle-tree-test_merkle.md

### [ACCEPT] key-range-merkle-diff-returns-key-names
`KeyRangeMerkleTree.diff_keys()` returns the string key names of divergent entries, translating numeric leaf indices back to keys
- Source: entries/2026/05/29/merkle-tree-test_merkle.md

### [REJECT] multi-leader-sync-snapshot-isolation
Duplicate of existing `multi-leader-sync-collects-then-distributes` which already captures that sync() drains all pending queues before distributing, preventing intra-round cascading
- Source: entries/2026/05/29/multi-leader-replication-multi_leader-MultiLeaderCluster.md

### [ACCEPT] multi-leader-ring-convergence-rounds
RING topology requires at least N−1 `sync()` rounds to fully propagate a single-source change across N nodes, because each round advances the change by exactly one hop
- Source: entries/2026/05/29/multi-leader-replication-multi_leader-MultiLeaderCluster.md

### [ACCEPT] multi-leader-convergence-skips-origin
`all_converged()` compares `(value, timestamp, is_tombstone)` but intentionally ignores `origin_node_id` (tuple index 2), since different arrival orders can record different origins for the same resolved state
- Source: entries/2026/05/29/multi-leader-replication-multi_leader-MultiLeaderCluster.md

### [ACCEPT] multi-leader-sync-count-includes-noops
The count returned by `sync()` includes idempotent no-op deliveries (changes already in the target's `_seen` set), so it reflects replication attempts, not accepted changes
- Source: entries/2026/05/29/multi-leader-replication-multi_leader-MultiLeaderCluster.md

### [ACCEPT] multi-leader-custom-merge-requires-merge-fn
Constructing a cluster with `CUSTOM_MERGE` strategy and `merge_fn=None` is accepted silently, but raises `TypeError` at the first actual conflict during `sync()`
- Source: entries/2026/05/29/multi-leader-replication-multi_leader-MultiLeaderCluster.md


## Proposed Beliefs

---

### Entry: multi-leader-replication/multi_leader.py — apply_remote_change

### [REJECT] apply-remote-change-idempotent
Duplicate of existing `multi-leader-idempotent-apply`
- Source: entries/2026/05/29/multi-leader-replication-multi_leader-apply_remote_change.md

### [REJECT] lww-tiebreak-uses-node-id
Duplicate of existing `multi-leader-lww-deterministic-tiebreak`
- Source: entries/2026/05/29/multi-leader-replication-multi_leader-apply_remote_change.md

### [REJECT] custom-merge-synthesizes-new-change
Duplicate of existing `multi-leader-custom-merge-new-timestamp`
- Source: entries/2026/05/29/multi-leader-replication-multi_leader-apply_remote_change.md

### [ACCEPT] conflict-requires-different-origins
A conflict in `apply_remote_change` is only detected when `local_origin != remote_node`; two writes from the same origin at different timestamps are treated as sequential updates and resolved by tuple comparison without generating a `ConflictRecord`.
- Source: entries/2026/05/29/multi-leader-replication-multi_leader-apply_remote_change.md

### [ACCEPT] pending-queue-drives-ring-propagation
Every accepted remote change (including synthesized merge results) is appended to `_pending` for downstream propagation, enabling multi-hop replication in ring topologies where a single sync round cannot reach all nodes.
- Source: entries/2026/05/29/multi-leader-replication-multi_leader-apply_remote_change.md

### [ACCEPT] apply-remote-change-no-merge-fn-guard
`apply_remote_change` does not validate that `merge_fn` is non-None when strategy is `CUSTOM_MERGE`; passing `None` raises `TypeError` at call time rather than a descriptive error.
- Source: entries/2026/05/29/multi-leader-replication-multi_leader-apply_remote_change.md

### [ACCEPT] apply-remote-change-advances-lamport-clock
`apply_remote_change` always advances the node's Lamport clock to `max(local_clock, remote_ts) + 1` before any conflict resolution, ensuring subsequent local writes have causally-later timestamps than any received change.
- Source: entries/2026/05/29/multi-leader-replication-multi_leader-apply_remote_change.md

### [ACCEPT] tombstone-reported-as-none-in-conflict
`ConflictRecord.remote_value` reports `None` for tombstoned changes rather than the internal `_TOMBSTONE` sentinel; callers see deletion as `None` and never observe the sentinel object.
- Source: entries/2026/05/29/multi-leader-replication-multi_leader-apply_remote_change.md

---

### Entry: multi-leader-replication/tester_test_multi_leader.py

### [REJECT] multi-leader-lww-tiebreak-uses-node-id
Duplicate of existing `multi-leader-lww-deterministic-tiebreak`
- Source: entries/2026/05/29/multi-leader-replication-tester_test_multi_leader.md

### [ACCEPT] multi-leader-custom-merge-idempotent-across-syncs
Custom merge functions are applied once per conflict; repeated sync rounds on already-converged values must not re-trigger the merge, preventing bugs like counters doubling on every sync cycle.
- Source: entries/2026/05/29/multi-leader-replication-tester_test_multi_leader.md

### [ACCEPT] multi-leader-ring-needs-multiple-rounds
Ring topology requires at least 2 sync rounds to propagate a write across 3+ nodes, unlike all-to-all topology which converges in a single round; `sync_until_converged()` loops to handle this.
- Source: entries/2026/05/29/multi-leader-replication-tester_test_multi_leader.md

### [ACCEPT] multi-leader-sync-is-discrete
Replication occurs only when `sync()` is explicitly called; there is no background replication thread or event-driven propagation, giving tests full control over replication timing.
- Source: entries/2026/05/29/multi-leader-replication-tester_test_multi_leader.md

### [ACCEPT] multi-leader-conflict-record-captures-both-values
`ConflictRecord` stores both `local_value` and `remote_value` along with the key and resolution strategy, providing a complete audit trail for every conflict resolution.
- Source: entries/2026/05/29/multi-leader-replication-tester_test_multi_leader.md

---

### Entry: raft-consensus/raft.py — _advance_commit_index

### [REJECT] raft-leader-only-commits-current-term
Duplicate of existing `raft-current-term-commit-only`
- Source: entries/2026/05/29/raft-consensus-raft-_advance_commit_index.md

### [ACCEPT] raft-commit-index-monotonic
`_commit_index` only ever increases within `_advance_commit_index` because the loop starts at `_commit_index + 1` and only assigns forward; it can never decrease or be reset.
- Source: entries/2026/05/29/raft-consensus-raft-_advance_commit_index.md

### [ACCEPT] raft-commit-requires-strict-majority
An entry is committed when `count > total // 2`, meaning for a 5-node cluster at least 3 replicas (including the leader) must have the entry, matching Raft's quorum requirement.
- Source: entries/2026/05/29/raft-consensus-raft-_advance_commit_index.md

### [ACCEPT] raft-advance-commit-on-tick-and-response
`_advance_commit_index` is invoked both on heartbeat ticks and immediately after successful append responses update `_match_index`, allowing eager commit advancement without waiting for the next heartbeat cycle.
- Source: entries/2026/05/29/raft-consensus-raft-_advance_commit_index.md

### [ACCEPT] raft-match-index-defaults-zero
Peers missing from `_match_index` are treated as having replicated nothing via `dict.get(peer, 0)`; `_become_leader` initializes all peers to 0, so this default is safe but allows graceful handling of unexpected state.
- Source: entries/2026/05/29/raft-consensus-raft-_advance_commit_index.md

---

### Entry: raft-consensus/test_raft.py

### [ACCEPT] raft-follower-rejects-writes
`RaftNode.client_request()` on a follower returns `{"success": False, "error": "not leader"}` as a structured dict rather than raising an exception; only the leader accepts client writes.
- Source: entries/2026/05/29/raft-consensus-test_raft.md

### [ACCEPT] raft-cluster-deterministic-simulation
`RaftCluster` uses tick-based deterministic simulation with no real clocks, threads, or network I/O; time advances only via explicit `tick(ms)` calls, making partition and election scenarios fully reproducible.
- Source: entries/2026/05/29/raft-consensus-test_raft.md

### [ACCEPT] raft-uncommitted-entries-overwritten
Uncommitted log entries from a deposed leader are overwritten when the node receives `AppendEntries` from a new leader with conflicting entries at the same index; only committed entries are durable.
- Source: entries/2026/05/29/raft-consensus-test_raft.md

### [ACCEPT] raft-minority-cannot-elect
A partition containing fewer than a strict majority of nodes cannot elect a leader; `get_leader()` returns `None` for a 2-of-5 minority partition.
- Source: entries/2026/05/29/raft-consensus-test_raft.md

### [ACCEPT] raft-terms-strictly-increase
Each new leader election produces a strictly higher term than the previous leader; terms never repeat or decrease across leader transitions, as validated by the `test_terms_monotonically_increasing` test.
- Source: entries/2026/05/29/raft-consensus-test_raft.md


### [ACCEPT] range-scan-no-mutation
`Partition.range_scan` is a pure read operation with no side effects — it never modifies `_keys`, `_values`, or any other state, and returns a fresh list the caller can mutate freely.
- Source: entries/2026/05/29/range-partitioning-range_partitioning-range_scan.md

### [ACCEPT] range-scan-relies-on-sorted-keys
`range_scan` correctness depends on `_keys` being sorted, an invariant maintained by `put`/`delete`/`split`/`merge` but not enforced structurally — direct mutation of `_keys` bypassing these methods silently produces wrong results.
- Source: entries/2026/05/29/range-partitioning-range_partitioning-range_scan.md

### [ACCEPT] range-scan-none-end-scans-unbounded
Passing `end=None` to `Partition.range_scan` sets the right boundary to `len(self._keys)`, scanning from `start` through the last key with no upper bound.
- Source: entries/2026/05/29/range-partitioning-range_partitioning-range_scan.md

### [ACCEPT] store-range-scan-passes-global-bounds
`RangePartitionedStore.range_scan` passes the same global `start_key`/`end_key` to every overlapping partition rather than clamping to partition boundaries, relying on `bisect_left` to naturally exclude out-of-range keys.
- Source: entries/2026/05/29/range-partitioning-range_partitioning-range_scan.md

### [REJECT] range-partition-scan-half-open
Duplicates proposed `range-scan-half-open-interval` above.
- Source: entries/2026/05/29/range-partitioning-test_range_partitioning.md

### [REJECT] range-partition-split-median
Duplicates existing belief `range-partitioning-split-at-median-index`.
- Source: entries/2026/05/29/range-partitioning-test_range_partitioning.md

### [REJECT] range-partition-boundaries-contiguous
Duplicates existing belief `range-partitioning-contiguous-half-open-coverage`.
- Source: entries/2026/05/29/range-partitioning-test_range_partitioning.md

### [ACCEPT] range-partition-no-exceptions-api
`RangePartitionedStore` uses return-value signaling: `get()` returns `None` for missing keys and `delete()` returns `False` for absent keys, rather than raising exceptions.
- Source: entries/2026/05/29/range-partitioning-test_range_partitioning.md

### [ACCEPT] range-partition-merge-guards-threshold
`merge_small_partitions()` only merges adjacent partitions when their combined size stays below `min_partition_size`, preventing immediate re-splitting after merge.
- Source: entries/2026/05/29/range-partitioning-test_range_partitioning.md

### [ACCEPT] replica-get-returns-tuple-or-none
`Replica.get` returns a `(value, version)` tuple when the key exists, or `None` when it doesn't — callers must guard against `None` before destructuring, as there is no sentinel tuple.
- Source: entries/2026/05/29/read-repair-read_repair-get.md

### [ACCEPT] replica-get-has-no-side-effects
`Replica.get` performs no mutations, availability checks, or I/O — it is a pure dictionary lookup, unlike `ReadRepairStore.get` which triggers read repair on stale replicas.
- Source: entries/2026/05/29/read-repair-read_repair-get.md

### [ACCEPT] replica-get-does-not-check-availability
`Replica.get` does not consult `self._available`; filtering unavailable replicas is the caller's responsibility via `ReadRepairStore._available_replicas()`.
- Source: entries/2026/05/29/read-repair-read_repair-get.md

### [ACCEPT] replica-versions-start-at-one
Versions assigned by `ReadRepairStore.put` start at 1 (computed as `max_version + 1` with initial max of 0), so a version of 0 in `ReadRepairStore.get` return values means "key not found," not "initial version."
- Source: entries/2026/05/29/read-repair-read_repair-get.md

### [REJECT] replica-put-rejects-stale-versions
Duplicates existing belief `replica-put-rejects-older-versions`.
- Source: entries/2026/05/29/read-repair-read_repair-put.md

### [REJECT] replica-put-return-value-unused
Duplicates existing belief `put-return-value-ignored-by-coordinator`.
- Source: entries/2026/05/29/read-repair-read_repair-put.md

### [ACCEPT] replica-put-no-concurrency-safety
The version check and store update in `Replica.put` are not atomic (TOCTOU race on lines 22–25); the method is unsafe under concurrent access without external locking.
- Source: entries/2026/05/29/read-repair-read_repair-put.md

### [ACCEPT] replica-versions-caller-coordinated
`Replica.put` does not generate version numbers; version assignment is the responsibility of `ReadRepairStore.put`, which reads the max version across all replicas (including unavailable ones) before writing.
- Source: entries/2026/05/29/read-repair-read_repair-put.md

### [ACCEPT] ddia-config-via-constructor-params
All 37 modules configure behavior entirely through constructor parameters (e.g., `max_file_size`, `sync_mode`, `page_size`) — there are no config files, environment variables, or settings modules.
- Source: entries/2026/05/29/repo-overview.md

### [ACCEPT] ddia-tester-files-stdout-validation
The `tester_test_*.py` files are designed as standalone scripts that print `"test_name PASSED"` / `"test_name FAILED"` to stdout and include `if __name__ == "__main__"` blocks, enabling automated verification without pytest.
- Source: entries/2026/05/29/repo-overview.md


### [ACCEPT] doc-partitioned-write-touches-one
`DocumentPartitionedDB.put()` always touches exactly 1 partition regardless of how many fields are indexed, because the secondary index is co-located with the document on its home partition.
- Source: entries/2026/05/29/secondary-index-partitioning-test_secondary_index_partitioning.md

### [REJECT] doc-partitioned-query-scatters-all
Duplicate of existing belief `doc-partitioned-query-touches-all`.
- Source: entries/2026/05/29/secondary-index-partitioning-test_secondary_index_partitioning.md

### [REJECT] term-partitioned-async-is-eventually-consistent
Substantially overlaps with existing belief `async-index-defers-mutations`, which already captures that mutations are queued and `flush_index()` must be called.
- Source: entries/2026/05/29/secondary-index-partitioning-test_secondary_index_partitioning.md

### [REJECT] both-strategies-clean-indexes-on-mutation
Duplicate of existing belief `secondary-index-stale-entry-cleanup`.
- Source: entries/2026/05/29/secondary-index-partitioning-test_secondary_index_partitioning.md

### [ACCEPT] operation-result-tracks-partition-cost
Every `DocumentPartitionedDB` and `TermPartitionedDB` operation returns an `OperationResult` with a `partitions_touched` field, making read/write amplification cost observable and directly comparable between strategies.
- Source: entries/2026/05/29/secondary-index-partitioning-test_secondary_index_partitioning.md

### [ACCEPT] secondary-index-no-error-on-missing
`get()` on a nonexistent key returns `OperationResult(data=None)` and `query_by_field()` for absent values returns empty results rather than raising exceptions; documents missing indexed fields are stored but not indexed.
- Source: entries/2026/05/29/secondary-index-partitioning-test_secondary_index_partitioning.md

### [ACCEPT] visibility-is-pure
`_is_visible` is a pure function with no side effects; it reads `_committed`, `_aborted`, `tx.active_at_start`, and version fields but never mutates any state.
- Source: entries/2026/05/29/snapshot-isolation-mvcc_database-_is_visible.md

### [ACCEPT] own-writes-always-visible
A transaction always sees versions it created itself (`created_by == tx.tx_id`), regardless of commit status, ensuring read-your-own-writes consistency within a transaction.
- Source: entries/2026/05/29/snapshot-isolation-mvcc_database-_is_visible.md

### [ACCEPT] aborted-writes-universally-invisible
A version created by an aborted transaction is invisible to every transaction, and a deletion by an aborted transaction is ignored — aborted transactions' effects are completely erased from all snapshots.
- Source: entries/2026/05/29/snapshot-isolation-mvcc_database-_is_visible.md

### [ACCEPT] deletion-visibility-is-symmetric
The same three-condition visibility rule (committed, not in active_at_start, lower tx_id) is applied independently to both the creating and deleting transaction of a version — `_is_visible` runs the same logic in two phases.
- Source: entries/2026/05/29/snapshot-isolation-mvcc_database-_is_visible.md

### [ACCEPT] mvcc-txid-monotonicity-assumed
The visibility check `created_by < tx.tx_id` assumes transaction IDs are monotonically increasing to mean "started before us"; non-monotonic or reused IDs would break snapshot correctness.
- Source: entries/2026/05/29/snapshot-isolation-mvcc_database-_is_visible.md

### [REJECT] commit-first-committer-wins
Duplicate of existing belief `first-committer-wins-on-write-conflict`.
- Source: entries/2026/05/29/snapshot-isolation-mvcc_database-commit.md

### [ACCEPT] commit-read-only-skip-conflict-check
Read-only transactions bypass write-write conflict detection entirely in `commit()` and always commit successfully, since they cannot produce write-write conflicts.
- Source: entries/2026/05/29/snapshot-isolation-mvcc_database-commit.md

### [ACCEPT] commit-timestamp-unused-by-visibility
Commit timestamps are assigned by `commit()` and stored in `_commit_timestamps`, but `_is_visible` never references them — visibility is determined purely by transaction IDs and the committed/active-at-start sets.
- Source: entries/2026/05/29/snapshot-isolation-mvcc_database-commit.md

### [ACCEPT] commit-aborts-on-conflict
On write-write conflict, `commit()` calls `abort(tx)` internally and returns `False`, so callers never need to manually abort after a failed commit; the transaction is dead after a `False` return.
- Source: entries/2026/05/29/snapshot-isolation-mvcc_database-commit.md

### [ACCEPT] commit-conflict-check-not-atomic
The conflict-check-then-commit sequence in `MVCCDatabase.commit()` is not atomic — no locking protects the window between checking versions and marking committed, requiring single-threaded execution for correctness.
- Source: entries/2026/05/29/snapshot-isolation-mvcc_database-commit.md

### [ACCEPT] commit-conflict-scans-all-versions
The write-write conflict check in `commit()` scans all versions of each key in the transaction's write set (not just the latest), making conflict detection O(versions) per written key.
- Source: entries/2026/05/29/snapshot-isolation-mvcc_database-commit.md

### [ACCEPT] gc-unconditionally-purges-aborted
`garbage_collect()` removes all versions with `created_by` in `_aborted` as its first step, regardless of active transaction state — aborted versions are invisible to everyone and always safe to drop.
- Source: entries/2026/05/29/snapshot-isolation-mvcc_database-garbage_collect.md

### [ACCEPT] gc-no-active-keeps-one
When no transactions are active, `garbage_collect()` retains at most one version per key — the latest committed non-deleted version — and drops everything else including fully-deleted keys.
- Source: entries/2026/05/29/snapshot-isolation-mvcc_database-garbage_collect.md

### [ACCEPT] gc-uses-min-txid-horizon
With active transactions, `garbage_collect()` uses `min(active tx_ids)` as the GC horizon; only versions superseded by a committed version with `tx_id < min_tx_id` are eligible for removal.
- Source: entries/2026/05/29/snapshot-isolation-mvcc_database-garbage_collect.md

### [ACCEPT] gc-not-thread-safe
`garbage_collect()` mutates `_versions` (replacing lists and deleting keys) without synchronization, assuming single-threaded execution — concurrent reads or writes during GC would race on the dict and its lists.
- Source: entries/2026/05/29/snapshot-isolation-mvcc_database-garbage_collect.md

### [ACCEPT] overlapping-uses-interval-overlap-test
`_overlapping` determines SSTable key-range overlap using the standard interval overlap test (`a.min_key <= b.max_key AND a.max_key >= b.min_key`) with lexicographic string comparison matching the SSTable sort order.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-_overlapping.md

### [ACCEPT] overlapping-is-pure-query
`_overlapping` performs no I/O, mutation, or side effects — it operates entirely on in-memory metadata (`min_key`, `max_key`) cached during `SSTableReader` construction.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-_overlapping.md

### [ACCEPT] lcs-overlap-selection-is-quadratic
During level N compaction, `_lcs_compact` calls `_overlapping` once per SSTable in the level to find the one with maximum overlap, then once more for the winner — making SSTable selection O(n*m) in level sizes.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-_overlapping.md

### [REJECT] empty-sstable-overlap-edge-case
Edge case trivia about empty SSTables defaulting to `min_key=""` — compaction never feeds empty SSTables so this never triggers in practice, making it untestable in the actual codebase.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-_overlapping.md


### [ACCEPT] stcs-merges-within-size-tiers
Size-tiered compaction only merges SSTables in the same size tier (determined by `_get_tier()` comparing `file_size` against `_size_thresholds`); it never merges a small SSTable directly with a much larger one.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-_stcs_compact.md

### [ACCEPT] stcs-min-threshold-default-four
A size tier must accumulate at least 4 SSTables (the default `_min_threshold`) before `_stcs_compact` merges them; buckets below this threshold are skipped entirely.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-_stcs_compact.md

### [REJECT] stcs-compact-preserves-tombstones
Duplicates `merge-preserves-tombstones-by-default` — `merge_sstables` defaults `remove_tombstones=False` regardless of caller.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-_stcs_compact.md

### [REJECT] stcs-compact-does-not-delete-old-files
Duplicates `compaction-manager-never-deletes-old-files` — already captured that CompactionManager orphans input files on disk after merging.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-_stcs_compact.md

### [ACCEPT] stcs-compact-mutates-sstable-list
`_stcs_compact` both removes old and appends new entries to `self._sstables` as a side effect; callers must not hold references into the list across a compaction call.
- Source: entries/2026/05/29/sstable-and-compaction-sstable-_stcs_compact.md

### [ACCEPT] stream-join-inner-requires-key-and-window-match
Inner join only produces a `JoinResult` when both key equality and `|t_left - t_right| <= window_duration` hold; either condition failing alone prevents a match.
- Source: entries/2026/05/29/stream-join-processor-test_stream_join_processor.md

### [REJECT] stream-join-left-miss-emitted-on-expiration
Duplicates `stream-join-miss-deferred` — unmatched events emit misses only when `advance_time` expires them, not eagerly.
- Source: entries/2026/05/29/stream-join-processor-test_stream_join_processor.md

### [ACCEPT] stream-join-late-events-dropped-silently
Events arriving after `watermark - allowed_lateness` are dropped and counted in `stats.late_events_dropped`; the processor never raises exceptions for late arrivals.
- Source: entries/2026/05/29/stream-join-processor-test_stream_join_processor.md

### [ACCEPT] stream-join-process-event-returns-immediate-matches
`process_event()` returns matches synchronously on each call (push-based) rather than batching to a flush boundary; results are available the instant a match is found.
- Source: entries/2026/05/29/stream-join-processor-test_stream_join_processor.md

### [ACCEPT] stream-join-buffers-bounded-by-window
The processor actively expires events so buffer sizes remain proportional to `window_duration × event_rate`, not total events processed; tests confirm buffers stay under 10 per side after 1000 events with a 5-second window.
- Source: entries/2026/05/29/stream-join-processor-test_stream_join_processor.md

### [ACCEPT] tumbling-window-aligned-to-zero
`TumblingWindowAggregator` aligns window boundaries to multiples of the window size starting from 0 — windows are `[0, size)`, `[size, 2*size)`, etc., not relative to the first event's timestamp.
- Source: entries/2026/05/29/stream-join-processor-test_stream_join_processor.md

### [ACCEPT] 2pc-blocking-window
The coordinator logs `"committing"` before any participant receives the commit decision, creating a window where a crash leaves participants locked indefinitely in the `"prepared"` state with no authority to self-resolve.
- Source: entries/2026/05/29/topic-2pc-blocking-problem.md

### [ACCEPT] participant-cannot-self-resolve
`Participant.recover()` returns a list of in-doubt transaction IDs but cannot commit or abort them; only the coordinator holds the decision, so participants remain locked until `Coordinator.recover()` re-sends it.
- Source: entries/2026/05/29/topic-2pc-blocking-problem.md

### [ACCEPT] recovery-requires-participant-availability
`Coordinator.recover()` checks `is_available()` before re-sending decisions and skips unavailable participants, meaning a double failure (coordinator crash + participant crash) leaves locks held until both are up and recovery re-runs.
- Source: entries/2026/05/29/topic-2pc-blocking-problem.md

### [ACCEPT] locks-block-future-transactions
A participant in the `"prepared"` state holds key-level locks (`self.locks[key] = tx_id`) that cause any subsequent transaction touching the same keys to abort with a lock conflict during its own `prepare()`.
- Source: entries/2026/05/29/topic-2pc-blocking-problem.md

### [ACCEPT] 2pc-recovery-single-pass
`Coordinator.recover()` scans the log once and re-sends decisions to currently-available participants; it does not retry or schedule follow-ups, so a participant still down at recovery time stays locked until `recover()` is manually called again.
- Source: entries/2026/05/29/topic-2pc-blocking-problem.md

### [REJECT] read-repair-only-fixes-queried-replicas
Duplicates `read-repair-scope-is-quorum-only` — already captured that `ReadRepairStore.get()` only repairs the R replicas selected for the quorum read.
- Source: entries/2026/05/29/topic-anti-entropy-vs-read-repair.md

### [ACCEPT] dynamo-read-repair-is-all-node
`DynamoCluster.get()` repairs all available nodes after every read, not just the quorum participants, trading higher per-read write fan-out for faster convergence compared to `ReadRepairStore`.
- Source: entries/2026/05/29/topic-anti-entropy-vs-read-repair.md

### [ACCEPT] merkle-diff-prunes-matching-subtrees
`MerkleTree._diff_recursive()` returns immediately when subtree hashes match, making the diff cost proportional to the number of divergent keys rather than total tree size — the logarithmic efficiency that makes anti-entropy practical.
- Source: entries/2026/05/29/topic-anti-entropy-vs-read-repair.md

### [ACCEPT] anti-entropy-repair-skips-unavailable
Both `ReadRepairStore.anti_entropy_repair()` and `DynamoCluster.anti_entropy_repair()` only scan and repair currently available replicas; a down replica must rely on a future anti-entropy run after it recovers.
- Source: entries/2026/05/29/topic-anti-entropy-vs-read-repair.md

### [ACCEPT] hinted-handoff-ttl-bounds-hint-lifetime
Hints expire after `created_at + ttl` (checked via `Hint.is_expired`); a node that stays down longer than the TTL window will not receive those hinted writes and must rely on anti-entropy for convergence.
- Source: entries/2026/05/29/topic-anti-entropy-vs-read-repair.md

### [REJECT] wal-no-undo-recovery
Duplicates `wal-is-redo-only` — already captured that the WAL implements only redo recovery with no undo phase.
- Source: entries/2026/05/29/topic-aries-three-pass-recovery.md

### [REJECT] wal-checkpoint-as-replay-fence
Duplicates `checkpoint-record-not-used-in-recovery` and `checkpoint-filtered-from-replay` — the checkpoint mechanism and its limitations are already captured.
- Source: entries/2026/05/29/topic-aries-three-pass-recovery.md

### [REJECT] mvcc-abort-is-logical-not-physical
Duplicates `abort-is-status-change-not-disk-rollback` — already captured that abort marks transaction status without reversing writes, relying on visibility checks to filter.
- Source: entries/2026/05/29/topic-aries-three-pass-recovery.md

### [REJECT] wal-batch-commit-atomicity
Duplicates `wal-batch-relies-on-practical-atomicity` and `wal-batch-single-write` — the single-buffer write with forced fsync and its non-guarantee are already captured.
- Source: entries/2026/05/29/topic-aries-three-pass-recovery.md


### [REJECT] async-index-defers-to-pending-queue
Duplicates existing belief `async-index-defers-mutations`
- Source: entries/2026/05/29/topic-async-index-consistency.md

### [ACCEPT] flush-index-drains-pending
`flush_index()` is the sole mechanism that materializes async index updates: it iterates `self._pending`, calls `_apply_index_op()` for each entry, and returns the count of operations applied
- Source: entries/2026/05/29/topic-async-index-consistency.md

### [ACCEPT] primary-key-reads-unaffected-by-async
`get(pk)` reads directly from `partitions[pid].documents` and is never affected by `async_index` mode — staleness only applies to `query_by_field` which reads from `global_index`
- Source: entries/2026/05/29/topic-async-index-consistency.md

### [REJECT] term-partition-query-touches-one-partition
Duplicates existing belief `term-partitioned-point-query-is-targeted`
- Source: entries/2026/05/29/topic-async-index-consistency.md

### [REJECT] sstable-writes-directly-to-final-path
Duplicates existing belief `no-atomic-file-creation`
- Source: entries/2026/05/29/topic-atomic-write-via-temp-rename.md

### [REJECT] lsm-flush-lacks-fsync
Duplicates existing beliefs `lsm-flush-double-loss-window` and `sstable-writer-finish-no-fsync`
- Source: entries/2026/05/29/topic-atomic-write-via-temp-rename.md

### [REJECT] wal-flush-is-python-only
Duplicates existing belief `lsm-wal-has-no-fsync`
- Source: entries/2026/05/29/topic-atomic-write-via-temp-rename.md

### [ACCEPT] bitcask-rename-is-rotation-not-safety
The `os.rename` calls in both Bitcask implementations rename active segments to frozen segments for namespace management, not for crash-safe atomic file creation — the file is already populated at its original path before the rename
- Source: entries/2026/05/29/topic-atomic-write-via-temp-rename.md

### [REJECT] sstable-header-rewrite-is-crash-vulnerable
Duplicates existing belief `sstable-writer-append-then-patch`
- Source: entries/2026/05/29/topic-atomic-write-via-temp-rename.md

### [ACCEPT] avro-block-encoding-zero-terminated
Arrays and maps are encoded as a sequence of count-prefixed blocks terminated by a 0-count block; counts and elements use zigzag+varint via `write_long`/`read_long`
- Source: entries/2026/05/29/topic-avro-block-encoding.md

### [ACCEPT] avro-negative-count-carries-byte-size
A negative block count `-N` in Avro's spec means N elements follow, preceded by a byte-size long that enables O(1) skipping of the entire block without parsing elements
- Source: entries/2026/05/29/topic-avro-block-encoding.md

### [ACCEPT] avro-encoder-positive-counts-only
The `AvroEncoder` always writes positive counts for array/map blocks, never negative counts with byte-size prefixes, trading skip efficiency for encoder simplicity
- Source: entries/2026/05/29/topic-avro-block-encoding.md

### [ACCEPT] avro-skip-is-recursive-not-bytejump
The decoder's `_skip` method discards unwanted fields by recursively walking the writer schema rather than using byte-size jumps, which works correctly but requires understanding the element schema
- Source: entries/2026/05/29/topic-avro-block-encoding.md

### [ACCEPT] leaf-nodes-store-all-values
Values are stored exclusively in leaf nodes; internal nodes contain only routing keys and child page pointers, making this a B+-tree not a B-tree
- Source: entries/2026/05/29/topic-b-plus-tree-leaf-chains.md

### [REJECT] leaf-sibling-chain-exists
Duplicates existing belief `btree-leaf-sibling-chain`
- Source: entries/2026/05/29/topic-b-plus-tree-leaf-chains.md

### [REJECT] range-scan-uses-leaf-chain
Duplicates existing belief `range-scan-follows-sibling-chain`
- Source: entries/2026/05/29/topic-b-plus-tree-leaf-chains.md

### [ACCEPT] point-lookup-always-reaches-leaf
Every `get` must descend to a leaf node regardless of tree height since internal nodes hold no values; cost is exactly *h* page reads for a tree of height *h*
- Source: entries/2026/05/29/topic-b-plus-tree-leaf-chains.md

### [REJECT] split-uses-key-count-not-bytes
Duplicates existing belief `btree-max-keys-forces-splits`
- Source: entries/2026/05/29/topic-b-tree-page-overflow.md

### [REJECT] single-entry-size-validated
Duplicates existing belief `single-kv-size-validated`
- Source: entries/2026/05/29/topic-b-tree-page-overflow.md

### [REJECT] leaf-serialization-is-variable-length
Subsumed by existing beliefs `leaf-wire-format-is-header-sibling-entries` and `no-page-size-enforcement-in-serialize`
- Source: entries/2026/05/29/topic-b-tree-page-overflow.md


### [ACCEPT] event-store-persist-no-flush-no-fsync
`EventStore._persist_event` opens the NDJSON file in append mode, writes one JSON line, and closes it without calling `flush()` or `os.fsync()`; written data may remain in OS buffers or Python buffers at crash time.
- Source: entries/2026/05/29/topic-batch-atomicity-gap.md

### [ACCEPT] event-store-persist-separate-open-close
`EventStore._persist_event` performs a separate `open()`/`write()`/`close()` cycle for each individual event, so `append_batch` with N events results in N independent file operations rather than a single buffered write.
- Source: entries/2026/05/29/topic-batch-atomicity-gap.md

### [ACCEPT] event-store-load-ignores-partial-batches
`EventStore._load_from_file` replays every NDJSON line on startup with no mechanism to detect or discard incomplete batches left by a mid-batch crash, unlike the WAL which uses `OP_COMMIT` records as transaction boundaries.
- Source: entries/2026/05/29/topic-batch-atomicity-gap.md

### [ACCEPT] btree-bisect-left-for-leaf-exact-match
Leaf lookups use `bisect_left` to find the leftmost insertion point, then check `keys[idx] == key` for exact match; `bisect_right` would return the position after all equal keys, overshooting the target.
- Source: entries/2026/05/29/topic-bisect-right-vs-left-in-btrees.md

### [ACCEPT] btree-internal-split-promotes-key
Internal node splits promote the midpoint key into the parent (removing it from the child), unlike leaf splits which copy it (the key remains in the right child); this asymmetry allows uniform `bisect_right` routing at all internal levels.
- Source: entries/2026/05/29/topic-bisect-right-vs-left-in-btrees.md

### [REJECT] event-store-append-batch-no-commit-marker
Duplicates existing belief `batch-append-not-crash-safe`.
- Source: entries/2026/05/29/topic-batch-atomicity-gap.md

### [REJECT] wal-append-batch-uses-commit-record
Duplicates existing beliefs `wal-batch-commit-sentinel` and `wal-batch-atomicity-via-commit-record`.
- Source: entries/2026/05/29/topic-batch-atomicity-gap.md

### [REJECT] event-store-persist-no-fsync
Subsumed by the more specific `event-store-persist-no-flush-no-fsync` proposed above.
- Source: entries/2026/05/29/topic-batch-atomicity-gap.md

### [REJECT] wal-batch-single-write-forced-sync
Duplicates existing beliefs `wal-batch-single-write` and `wal-batch-always-fsyncs`.
- Source: entries/2026/05/29/topic-batch-atomicity-gap.md

### [REJECT] wal-sync-mode-flush-plus-fsync
Duplicates existing belief `standalone-wal-always-pairs-flush-fsync`.
- Source: entries/2026/05/29/topic-batch-sync-mode.md

### [REJECT] wal-batch-sync-counter-reset
Duplicates existing belief `batch-mode-defers-fsync` which covers the batch sync deferral mechanism.
- Source: entries/2026/05/29/topic-batch-sync-mode.md

### [REJECT] wal-none-mode-no-sync
Duplicates existing belief `none-mode-never-syncs-on-append`.
- Source: entries/2026/05/29/topic-batch-sync-mode.md

### [REJECT] wal-force-sync-overrides-mode
Covered by existing beliefs `wal-batch-always-fsyncs` (the effect) and `wal-do-sync-branches-mutually-exclusive` (the mechanism).
- Source: entries/2026/05/29/topic-batch-sync-mode.md

### [REJECT] wal-append-respects-sync-mode
Covered collectively by existing beliefs `wal-fsync-per-entry`, `batch-mode-defers-fsync`, and `none-mode-never-syncs-on-append`.
- Source: entries/2026/05/29/topic-batch-sync-mode.md

### [REJECT] sbt-db-is-riak-default
General BEAM VM knowledge about Riak production conventions, not a testable claim about the target codebase. The entry itself notes it "draws on general BEAM VM knowledge rather than code observations in the target repository."
- Source: entries/2026/05/29/topic-beam-scheduler-binding.md

### [REJECT] scheduler-binding-benefits-ets-writes-more-than-file-reads
General systems knowledge about CPU cache behavior, not a claim about code in the target repository.
- Source: entries/2026/05/29/topic-beam-scheduler-binding.md

### [REJECT] numa-creates-ets-access-asymmetry
General systems knowledge about NUMA memory architecture, not grounded in target codebase observations.
- Source: entries/2026/05/29/topic-beam-scheduler-binding.md

### [REJECT] unbound-schedulers-cause-cache-migration
General systems knowledge about OS thread scheduling, not a codebase-specific claim.
- Source: entries/2026/05/29/topic-beam-scheduler-binding.md

### [REJECT] btree-bisect-right-matches-copy-up-split
Duplicates existing belief `find-leaf-bisect-right-consistency` which already captures the causal link: "consistent with the split strategy where the separator key equals the first key of the right sibling."
- Source: entries/2026/05/29/topic-bisect-right-vs-left-in-btrees.md

### [REJECT] btree-split-asymmetry-leaf-vs-internal
The leaf-copy side duplicates `btree-leaf-split-copies-key`; the internal-promote side is captured in the accepted `btree-internal-split-promotes-key` above.
- Source: entries/2026/05/29/topic-bisect-right-vs-left-in-btrees.md

### [REJECT] btree-bisect-left-internal-would-misroute
Counterfactual reasoning about what would break under a hypothetical code change, not a testable factual claim about the codebase as it exists.
- Source: entries/2026/05/29/topic-bisect-right-vs-left-in-btrees.md

### [REJECT] lsm-wal-missing-fsync
Duplicates existing belief `lsm-wal-has-no-fsync`.
- Source: entries/2026/05/29/topic-bitcask-durability-guarantees.md

### [REJECT] btree-wal-fsync-before-data-write
Duplicates existing beliefs `btree-wal-before-data` and `btree-wal-fsync-per-entry`.
- Source: entries/2026/05/29/topic-bitcask-durability-guarantees.md

### [REJECT] standalone-wal-always-pairs-flush-fsync
Already in the accepted beliefs list.
- Source: entries/2026/05/29/topic-bitcask-durability-guarantees.md

### [REJECT] bitcask-fsync-is-configurable
Duplicates existing belief `hash-index-sync-writes-controls-fsync`.
- Source: entries/2026/05/29/topic-bitcask-durability-guarantees.md

### [REJECT] btree-page-writes-not-individually-fsynced
Duplicates existing belief `btree-wal-no-data-fsync`.
- Source: entries/2026/05/29/topic-bitcask-durability-guarantees.md


### [ACCEPT] wal-batch-sync-force-override
The WAL's `_do_sync` batch mode (fsync every N writes) is overridden by `force=True` in `append_batch` and `checkpoint`, ensuring atomic batch boundaries and checkpoint records are always fsynced regardless of configured sync mode.
- Source: entries/2026/05/29/topic-bitcask-durability-tradeoffs.md

### [ACCEPT] event-store-no-explicit-durability
`event-sourcing-store/event_store.py:_persist_event` relies solely on Python's context manager `close()` for implicit `flush()` with no explicit `flush()` or `fsync()`, providing no durability guarantee against either process crash or power loss.
- Source: entries/2026/05/29/topic-bitcask-durability-tradeoffs.md

### [REJECT] bitcask-flush-always-fsync-conditional
Duplicate of existing `hash-index-sync-writes-controls-fsync` which already captures that `sync_writes` controls whether each write fsyncs to disk.
- Source: entries/2026/05/29/topic-bitcask-durability-tradeoffs.md

### [REJECT] lsm-wal-missing-fsync
Duplicate of existing `lsm-wal-has-no-fsync` which already captures that the LSM WAL never calls fsync.
- Source: entries/2026/05/29/topic-bitcask-durability-tradeoffs.md

### [REJECT] btree-wal-syncs-data-file-defers
Covered by existing `btree-wal-fsync-per-entry` (WAL fsyncs every log entry) and `btree-wal-no-data-fsync` (data file defers fsync).
- Source: entries/2026/05/29/topic-bitcask-durability-tradeoffs.md

### [ACCEPT] lsm-compaction-blocks-callers
`LSMTree.compact()` runs synchronously in the caller's thread, meaning a `put()` or `delete()` that triggers the SSTable count threshold will block until the full k-way merge completes.
- Source: entries/2026/05/29/topic-bitcask-merge-window-scheduling.md

### [ACCEPT] no-compaction-rate-limiting
Neither LSM implementation throttles compaction I/O throughput; a large merge will saturate disk bandwidth without regard to concurrent foreground reads or writes.
- Source: entries/2026/05/29/topic-bitcask-merge-window-scheduling.md

### [ACCEPT] compaction-strategy-vs-scheduling-decoupled
The `sstable.py` `CompactionManager` separates *which* SSTables to merge (size-tiered vs. leveled strategy) from *when* to merge, but neither the manager nor any caller implements scheduling, rate-limiting, or load-aware deferral logic.
- Source: entries/2026/05/29/topic-bitcask-merge-window-scheduling.md

### [REJECT] compaction-trigger-is-count-only
Duplicate of existing `compaction-triggered-by-sstable-count` which already captures that compaction triggers on SSTable count thresholds.
- Source: entries/2026/05/29/topic-bitcask-merge-window-scheduling.md

### [ACCEPT] neither-bitcask-supports-expiry
Neither Bitcask implementation supports per-key TTL or expiry; `hash-index-storage` stores a timestamp in the header but never checks it for expiration, and `log-structured-hash-table` omits timestamps entirely.
- Source: entries/2026/05/29/topic-bitcask-paper-vs-implementation.md

### [REJECT] hash-index-storage-no-crc
Duplicate of existing `hash-index-no-crc-by-design` and `hash-index-bitcask-no-checksum`.
- Source: entries/2026/05/29/topic-bitcask-paper-vs-implementation.md

### [REJECT] log-structured-bitcask-has-crc
Duplicate of existing `bitcask-crc32-raises-corruption-error` and `bitcask-crc-validated-on-read-only`.
- Source: entries/2026/05/29/topic-bitcask-paper-vs-implementation.md

### [REJECT] hash-index-no-auto-compact
Duplicate of existing `hash-index-compaction-manual-only`.
- Source: entries/2026/05/29/topic-bitcask-paper-vs-implementation.md

### [REJECT] empty-string-tombstone-bug
Duplicate of existing `bitcask-tombstone-is-empty-string` and `bitcask-tombstone-semantics` which already capture the empty-string sentinel and its implications.
- Source: entries/2026/05/29/topic-bitcask-paper-vs-implementation.md

### [REJECT] hash-index-tombstone-is-zero-length-value
Duplicate of existing `bitcask-tombstone-is-empty-string` which already captures that tombstones are zero-length/empty-string values.
- Source: entries/2026/05/29/topic-bitcask-tombstone-lifecycle.md

### [REJECT] log-structured-tombstone-is-sentinel-bytes
Duplicate of existing `log-structured-sentinel-is-in-band` which already captures the sentinel byte tombstone encoding.
- Source: entries/2026/05/29/topic-bitcask-tombstone-lifecycle.md

### [REJECT] hash-index-hint-files-inline-with-compaction
Duplicate of existing `hint-written-only-during-compaction`.
- Source: entries/2026/05/29/topic-bitcask-tombstone-lifecycle.md

### [REJECT] log-structured-hint-files-decoupled-from-compaction
Duplicate of existing `log-structured-hint-creation-decoupled` and `log-structured-hint-creation-is-manual`.
- Source: entries/2026/05/29/topic-bitcask-tombstone-lifecycle.md

### [REJECT] sentinel-tombstone-lacks-put-guard
Duplicate of existing `put-no-tombstone-guard`.
- Source: entries/2026/05/29/topic-bitcask-tombstone-lifecycle.md

### [ACCEPT] sstable-block-size-is-index-interval
`SSTableWriter`'s `block_size` parameter controls the sparse index frequency (one index entry per N records), not a physical byte-aligned block size; records are written contiguously with no padding or alignment to fixed byte boundaries.
- Source: entries/2026/05/29/topic-block-based-wal-format.md

### [ACCEPT] no-fixed-block-boundaries-in-sstable
Both SSTable implementations write variable-length records contiguously with no fixed-size block structure, making mid-file resynchronization after corruption impossible — a corrupted length field causes the reader to consume all subsequent valid records as garbage.
- Source: entries/2026/05/29/topic-block-based-wal-format.md

### [REJECT] wal-corruption-stops-replay
Duplicate of existing `wal-corruption-stops-per-file` and `wal-crc-mismatch-halts-all-replay`.
- Source: entries/2026/05/29/topic-block-based-wal-format.md

### [REJECT] wal-has-per-record-checksums
Duplicate of existing `wal-crc-includes-op-type` and `crc32-covers-records-not-files`.
- Source: entries/2026/05/29/topic-block-based-wal-format.md


### [ACCEPT] standard-bloom-suffices-for-immutable-sstables
Standard bloom filters lack deletion support, but SSTables are write-once-then-discarded, so deletion is never needed and the 8x space savings over counting filters is free
- Source: entries/2026/05/29/topic-bloom-filter-on-sstables.md

### [REJECT] counting-bloom-8x-overhead
Duplicates existing belief `cbf-8x-memory-vs-standard`
- Source: entries/2026/05/29/topic-bloom-filter-on-sstables.md

### [REJECT] bloom-filter-not-integrated-in-lsm
Duplicates existing belief `bloom-filter-not-integrated`
- Source: entries/2026/05/29/topic-bloom-filter-on-sstables.md

### [REJECT] bloom-serialization-enables-sstable-embedding
Duplicates existing belief `bloom-serialization-footer-ready`
- Source: entries/2026/05/29/topic-bloom-filter-on-sstables.md

### [REJECT] btree-no-concurrency-control
Duplicates existing belief `btree-single-threaded-assumption`
- Source: entries/2026/05/29/topic-btree-concurrency-control.md

### [ACCEPT] wal-has-threading-lock
`WriteAheadLog` in `wal.py:80` uses a `threading.Lock` to serialize `append`, `append_batch`, `checkpoint`, and `truncate`, making it the only concurrency-aware component in the codebase
- Source: entries/2026/05/29/topic-btree-concurrency-control.md

### [REJECT] leaf-next-sibling-exists
Covered by existing beliefs `leaf-split-updates-sibling-pointers` and `sibling-field-is-structural-only`
- Source: entries/2026/05/29/topic-btree-concurrency-control.md

### [REJECT] allocate-page-race-condition
Direct consequence of existing belief `btree-single-threaded-assumption`; the race is just one manifestation of the general no-synchronization fact
- Source: entries/2026/05/29/topic-btree-concurrency-control.md

### [REJECT] btree-no-merge-or-rebalance
Duplicates existing belief `btree-no-underflow-rebalancing`
- Source: entries/2026/05/29/topic-btree-delete-underflow-handling.md

### [REJECT] btree-free-list-for-page-reuse
Covered by existing beliefs `btree-free-list-intrusive` and `free-list-layout-coupled`
- Source: entries/2026/05/29/topic-btree-delete-underflow-handling.md

### [REJECT] btree-underfull-leaves-persist
Direct consequence of existing belief `btree-no-underflow-rebalancing`; the space amplification follows mechanically from the lack of merge/rebalance
- Source: entries/2026/05/29/topic-btree-delete-underflow-handling.md

### [REJECT] bully-sends-election-to-higher-only
Covered by existing belief `bully-highest-id-wins` which already states "start_election only sends ELECTION to higher-ID nodes"
- Source: entries/2026/05/29/topic-bully-vs-ring-election.md

### [ACCEPT] bully-cascade-causes-quadratic-messages
Receiving an ELECTION from a lower-ID node triggers both an ALIVE response and a new `start_election` call, creating the O(n²) message cascade in worst-case elections
- Source: entries/2026/05/29/topic-bully-vs-ring-election.md

### [ACCEPT] bully-coordinator-is-broadcast
`declare_victory` sends a COORDINATOR message to every other node in the cluster (O(n) messages), not just to nodes that participated in the election
- Source: entries/2026/05/29/topic-bully-vs-ring-election.md

### [ACCEPT] bully-assumes-full-connectivity
The Bully Algorithm requires any node to message any other node directly via `all_node_ids`; there is no ring or tree overlay topology
- Source: entries/2026/05/29/topic-bully-vs-ring-election.md

### [REJECT] bully-highest-available-always-wins
Duplicates existing belief `bully-highest-id-wins`
- Source: entries/2026/05/29/topic-bully-vs-ring-election.md

### [ACCEPT] live-projection-subscribes-before-catchup
`LiveProjection.__init__` registers the subscriber before the user calls `catch_up()`, making duplicate processing possible but event loss impossible in the current single-threaded design
- Source: entries/2026/05/29/topic-catch-up-subscription-gap.md

### [REJECT] no-concurrency-control-in-event-store
Covered by existing belief `no-concurrency-primitives`
- Source: entries/2026/05/29/topic-catch-up-subscription-gap.md

### [REJECT] subscriber-notification-is-synchronous
Duplicates existing belief `subscriber-dispatch-is-synchronous`
- Source: entries/2026/05/29/topic-catch-up-subscription-gap.md

### [ACCEPT] catch-up-uses-event-id-filtering
`catch_up()` calls `read_all(from_position=self._position + 1)` to skip already-processed events, but `_on_event` has no equivalent position guard — an asymmetry that enables duplicate processing
- Source: entries/2026/05/29/topic-catch-up-subscription-gap.md


### [ACCEPT] cdc-write-and-log-are-synchronously-coupled
Every CDCDatabase mutation (insert/update/delete) writes to in-memory state and appends to CDCLog in the same synchronous call with no failure boundary between them
- Source: entries/2026/05/29/topic-cdc-flush-semantics.md

### [ACCEPT] cdc-consumer-position-is-volatile
CDCConsumer tracks its read position as an in-memory integer (`_position`) with no durable offset storage, meaning position is lost on process restart
- Source: entries/2026/05/29/topic-cdc-flush-semantics.md

### [ACCEPT] cdc-log-compact-keeps-latest-per-key
CDCLog.compact() retains only the most recent event per (table, key) pair, matching Kafka log compaction semantics but without concurrent-reader safety
- Source: entries/2026/05/29/topic-cdc-flush-semantics.md

### [ACCEPT] unbundled-db-is-log-first-cdc-is-state-first
The unbundled database writes to WAL before storage engine, while the CDC module writes to in-memory rows before appending the log — opposite ordering of the source-of-truth relationship
- Source: entries/2026/05/29/topic-cdc-flush-semantics.md

### [ACCEPT] tester-files-are-hand-written
The `tester_test_*.py` files contain no auto-generation markers and are hand-authored standalone test suites, not generated by code-expert or any other pipeline — contradicts existing `tester-files-are-generated-artifacts`
- Source: entries/2026/05/29/topic-code-expert-generation-pipeline.md

### [REJECT] tester-stdout-protocol
Duplicates existing belief `tester-files-use-stdout-validation`
- Source: entries/2026/05/29/topic-code-expert-generation-pipeline.md

### [ACCEPT] tester-test-decomposition
Tester files decompose monolithic pytest tests into smaller focused functions with docstrings (e.g., `test_basic` becomes `test_basic_put_get` + `test_range_scan` + `test_persistence`)
- Source: entries/2026/05/29/topic-code-expert-generation-pipeline.md

### [ACCEPT] code-expert-has-no-codegen
The code-expert workflow is read-only against the target repository — it scans, explores, and extracts beliefs but never generates or modifies source files
- Source: entries/2026/05/29/topic-code-expert-generation-pipeline.md

### [REJECT] combiner-runs-per-mapper-partition
Duplicates existing belief `mr-combiner-is-map-side-only` which already states the combiner runs inside `_run_mapper` on each mapper's output for a single partition independently
- Source: entries/2026/05/29/topic-combiner-correctness.md

### [ACCEPT] combiner-not-type-checked
The combiner's callable signature matches the reducer's, but no runtime or static check verifies associativity or commutativity — a non-associative combiner (e.g., average) produces silently wrong results that vary with `num_mappers`
- Source: entries/2026/05/29/topic-combiner-correctness.md

### [ACCEPT] combiner-does-not-affect-map-output-stats
`map_output_records` is incremented before the combiner runs (line 108), so stats reflect pre-combiner volume; the test at line 68 asserts this explicitly
- Source: entries/2026/05/29/topic-combiner-correctness.md

### [ACCEPT] combiner-output-indistinguishable-from-mapper-output
The reducer reads combiner output from intermediate JSON files identically to raw mapper output, with no metadata distinguishing pre-aggregated values from raw emitted pairs
- Source: entries/2026/05/29/topic-combiner-correctness.md

### [REJECT] wal-replay-is-flat
Duplicates existing belief `wal-replay-ignores-commit` which already states replay returns all PUT/DELETE records regardless of COMMIT presence
- Source: entries/2026/05/29/topic-commit-aware-replay-design.md

### [REJECT] wal-batch-atomicity-via-single-write
Duplicates existing belief `wal-batch-single-write` which already describes the single-bytearray write pattern
- Source: entries/2026/05/29/topic-commit-aware-replay-design.md

### [REJECT] wal-corruption-stops-replay
Duplicates existing belief `wal-crc-mismatch-halts-all-replay`
- Source: entries/2026/05/29/topic-commit-aware-replay-design.md

### [ACCEPT] wal-no-nested-batches
The WAL format has no batch-start marker and no nesting support; the lock held during `append_batch` prevents interleaving, making nested or concurrent batches structurally impossible
- Source: entries/2026/05/29/topic-commit-aware-replay-design.md

### [REJECT] wal-replay-ignores-commit-markers
Duplicates existing belief `wal-replay-ignores-commit`
- Source: entries/2026/05/29/topic-commit-record-replay-semantics.md

### [REJECT] wal-crash-safety-via-truncation-detection
Covered by existing beliefs `wal-truncation-vs-corruption-distinction` and `wal-partial-read-is-eof` which already describe the two failure modes and their handling
- Source: entries/2026/05/29/topic-commit-record-replay-semantics.md

### [REJECT] wal-no-begin-marker-prevents-batch-grouping
Substantially overlaps with accepted `wal-no-nested-batches` and existing `replay-does-not-enforce-batch-atomicity`
- Source: entries/2026/05/29/topic-commit-record-replay-semantics.md

### [REJECT] wal-docstring-implementation-mismatch
Duplicates existing belief `wal-docstring-describes-intent-not-behavior`
- Source: entries/2026/05/29/topic-commit-record-replay-semantics.md


### [REJECT] merge-produces-single-file
Duplicate of existing `lcs-produces-single-output-sstable` which already states merge produces one SSTable without size-based splitting
- Source: entries/2026/05/29/topic-compaction-output-splitting.md

### [REJECT] lcs-l1-can-have-overlapping-ranges
Duplicate of existing `lcs-produces-single-output-sstable` which already notes L1+ will have overlapping key ranges
- Source: entries/2026/05/29/topic-compaction-output-splitting.md

### [REJECT] merge-resolves-duplicates-by-timestamp
Duplicate of existing `sstable-merge-dedup-newest-timestamp`
- Source: entries/2026/05/29/topic-compaction-output-splitting.md

### [ACCEPT] sstable-writer-single-file-lifecycle
`SSTableWriter` binds to one file descriptor at construction with no method to rotate to a new file mid-write, making it structurally incompatible with size-bounded output splitting
- Source: entries/2026/05/29/topic-compaction-output-splitting.md

### [ACCEPT] compaction-result-assigned-level-1
`CompactionManager.run_compaction` with leveled strategy assigns merged output to level 1 but does not verify that the new file's key range is disjoint from existing L1 files
- Source: entries/2026/05/29/topic-compaction-output-splitting.md

### [REJECT] btree-no-concurrency-control
Duplicate of existing `btree-single-threaded-assumption`
- Source: entries/2026/05/29/topic-concurrency-and-latch-coupling.md

### [ACCEPT] leaf-chain-correct-single-threaded
The `next_sibling` leaf chain is correctly maintained through splits because the WAL logs the right page before the left page (ensuring the pointer target is durable before anything references it) and no concurrent reader can observe intermediate states
- Source: entries/2026/05/29/topic-concurrency-and-latch-coupling.md

### [ACCEPT] lehman-yao-needs-high-keys-and-internal-links
The existing leaf `next_sibling` pointer is necessary but insufficient for Lehman-Yao concurrent access; the algorithm also requires right-links on internal nodes and a high key per node, neither of which this implementation has
- Source: entries/2026/05/29/topic-concurrency-and-latch-coupling.md

### [REJECT] concurrent-scan-split-can-skip-keys
Theoretical failure mode about concurrent access that cannot occur in the single-threaded implementation; not a testable claim about the code as it exists
- Source: entries/2026/05/29/topic-concurrency-and-latch-coupling.md

### [REJECT] lsm-flush-not-using-immutable-list
Duplicate of existing `flush-skips-immutable-memtable-stage`
- Source: entries/2026/05/29/topic-concurrent-flush-safety.md

### [REJECT] lsm-read-path-checks-immutable-memtables
Duplicate of existing `read-path-anticipates-concurrent-flush`
- Source: entries/2026/05/29/topic-concurrent-flush-safety.md

### [ACCEPT] lsm-memtable-swap-is-reference-not-copy
`_flush` line 307 `frozen = self._memtable` captures a Python reference to the SortedDict, not a deep copy, so concurrent mutation of the dict after the swap would be unsafe without synchronization
- Source: entries/2026/05/29/topic-concurrent-flush-safety.md

### [REJECT] lsm-flush-synchronous-invariant
Duplicate of existing `single-threaded-masks-immutable-gap` which already captures that single-threaded execution is what makes the flush-without-immutable-memtable pattern safe
- Source: entries/2026/05/29/topic-concurrent-flush-safety.md

### [ACCEPT] strategy-enum-covers-two-of-three
`ConflictStrategy` implements LWW and custom merge but not CRDTs; the CRDT module exists separately in `crdts.py` with no integration into the multi-leader replication strategy dispatch
- Source: entries/2026/05/29/topic-conflict-resolution-strategies.md

### [REJECT] lww-tiebreak-uses-node-id
Duplicate of existing `multi-leader-lww-deterministic-tiebreak`
- Source: entries/2026/05/29/topic-conflict-resolution-strategies.md

### [REJECT] custom-merge-emits-new-timestamp
Duplicate of existing `multi-leader-custom-merge-new-timestamp`
- Source: entries/2026/05/29/topic-conflict-resolution-strategies.md

### [ACCEPT] no-vector-clocks-anywhere
The entire repository uses scalar Lamport clocks for ordering; no vector clock or version vector implementation exists, so true concurrency detection (distinguishing causal order from concurrent writes) is absent
- Source: entries/2026/05/29/topic-conflict-resolution-strategies.md

### [ACCEPT] crdts-are-self-resolving
Each CRDT type in `crdts.py` (GCounter, PNCounter, LWWRegister, ORSet) encodes conflict resolution in its own `merge()` method, requiring no external strategy enum or callback
- Source: entries/2026/05/29/topic-conflict-resolution-strategies.md

### [ACCEPT] encode-with-id-no-magic-byte
`SchemaRegistry.encode_with_id` uses a raw 4-byte big-endian schema ID prefix with no magic byte, unlike Confluent's 5-byte header (0x00 + 4-byte ID), so there is no format versioning for future wire format changes
- Source: entries/2026/05/29/topic-confluent-schema-registry-protocol.md

### [ACCEPT] decode-with-id-no-format-validation
`decode_with_id` does not validate the message format before parsing; any 4+ byte input will be interpreted as schema-ID-prefixed data, producing a confusing KeyError on invalid input rather than a clear format error
- Source: entries/2026/05/29/topic-confluent-schema-registry-protocol.md

### [ACCEPT] schema-registry-in-memory-only
`SchemaRegistry` is a pure in-memory dict with monotonic ID assignment, with no persistence, HTTP API, or subject-based grouping
- Source: entries/2026/05/29/topic-confluent-schema-registry-protocol.md

### [ACCEPT] schema-registry-no-compat-enforcement
Schema registration accepts any schema without checking compatibility against previously registered versions under the same subject, unlike Confluent which enforces backward/forward/full compatibility on registration
- Source: entries/2026/05/29/topic-confluent-schema-registry-protocol.md


### [ACCEPT] pytest-must-run-per-directory
Tests must be executed from within each module's directory (e.g., `cd bloom-filter && pytest`) because imports use bare module names with no package prefix or installed package
- Source: entries/2026/05/29/topic-conftest-and-pytest-config.md

### [REJECT] no-root-pytest-config
Duplicate of existing belief `no-pytest-config-exists`
- Source: entries/2026/05/29/topic-conftest-and-pytest-config.md

### [REJECT] modules-are-standalone-islands
Duplicate of existing belief `ddia-modules-self-contained`
- Source: entries/2026/05/29/topic-conftest-and-pytest-config.md

### [REJECT] no-shared-fixtures-exist
Covered by existing beliefs `no-pytest-config-exists` and `ddia-modules-self-contained`
- Source: entries/2026/05/29/topic-conftest-and-pytest-config.md

### [ACCEPT] no-conftest-anywhere
The ddia-implementations repo contains zero `conftest.py` files at any directory level — confirmed by exhaustive grep across all 37 modules, not just root
- Source: entries/2026/05/29/topic-conftest-hierarchy.md

### [REJECT] no-pytest-config-files
Duplicate of existing belief `no-pytest-config-exists`
- Source: entries/2026/05/29/topic-conftest-hierarchy.md

### [REJECT] no-collection-customization
Duplicate of existing belief `tester-test-files-not-excluded` which already states no `collect_ignore`, `norecursedirs`, `testpaths`, or `python_files` directives exist
- Source: entries/2026/05/29/topic-conftest-hierarchy.md

### [REJECT] modules-share-no-fixtures
Implied by existing beliefs `ddia-modules-self-contained` and `no-pytest-config-exists`
- Source: entries/2026/05/29/topic-conftest-hierarchy.md

### [REJECT] tester-files-not-excluded
Duplicate of existing belief `tester-test-files-not-excluded`
- Source: entries/2026/05/29/topic-conftest-hierarchy.md

### [REJECT] counting-bloom-saturated-counters-are-permanent
Duplicate of existing beliefs `cbf-saturated-counters-are-permanent` and `cbf-saturated-counters-never-decrement`
- Source: entries/2026/05/29/topic-counter-overflow-and-cuckoo-filters.md

### [REJECT] counting-bloom-4bit-default
Already captured in existing belief `cbf-saturated-counters-never-decrement` which states "default 15 for 4-bit counters"
- Source: entries/2026/05/29/topic-counter-overflow-and-cuckoo-filters.md

### [REJECT] counting-bloom-byte-per-counter-waste
Duplicate of existing beliefs `cbf-one-byte-per-counter` and `cbf-8x-memory-vs-standard`
- Source: entries/2026/05/29/topic-counter-overflow-and-cuckoo-filters.md

### [ACCEPT] no-cuckoo-filter-in-codebase
The repository contains no cuckoo filter implementation; the counting Bloom filter is the only deletion-supporting probabilistic filter, leaving the saturation problem unsolved
- Source: entries/2026/05/29/topic-counter-overflow-and-cuckoo-filters.md

### [ACCEPT] saturation-boundary-untested
No test in `test_bloom_filter.py` exercises the counter saturation boundary (16+ collisions at a single position) to verify deletion correctness under overflow
- Source: entries/2026/05/29/topic-counter-overflow-and-cuckoo-filters.md

### [ACCEPT] counting-bloom-4bit-safe-at-capacity
At designed capacity (n ≤ expected_items), the expected number of saturated 4-bit counters is effectively zero because the per-position load follows Poisson(ln 2 ≈ 0.693), making P(count ≥ 15) ≈ 10⁻¹⁵
- Source: entries/2026/05/29/topic-counter-saturation-probability.md

### [REJECT] counting-bloom-saturation-breaks-remove
Duplicate of existing belief `cbf-saturated-counters-are-permanent`
- Source: entries/2026/05/29/topic-counter-saturation-probability.md

### [REJECT] counting-bloom-wasted-bits
Duplicate of existing belief `cbf-one-byte-per-counter`
- Source: entries/2026/05/29/topic-counter-saturation-probability.md

### [ACCEPT] counting-bloom-overload-threshold
At 10× overload (10× expected_items inserted), roughly 0.5% of CountingBloomFilter counters saturate; at 20× overload, approximately half saturate, effectively degrading the filter to a non-counting Bloom filter
- Source: entries/2026/05/29/topic-counter-saturation-probability.md

### [ACCEPT] projection-is-stateless-replay
A `Projection`'s `_state` is a cache of derived data, not a source of truth — it can be fully reconstructed from the event log at any time by replaying from position 0
- Source: entries/2026/05/29/topic-cqrs-read-models.md

### [REJECT] live-projection-uses-subscriber-push
Duplicate of existing beliefs `live-projection-uses-subscriber-list` and `es-live-projection-auto-updates`
- Source: entries/2026/05/29/topic-cqrs-read-models.md

### [REJECT] snapshot-captures-position-and-state
Duplicate of existing belief `es-snapshot-saves-state-and-position`
- Source: entries/2026/05/29/topic-cqrs-read-models.md

### [ACCEPT] cdc-and-event-sourcing-share-projection-pattern
`CDCConsumer`/`MaterializedView` in `cdc.py` and `Projection` in `event_store.py` implement the same structural pattern — track position, pull new events, apply handlers — against different event sources (database mutations vs. domain events)
- Source: entries/2026/05/29/topic-cqrs-read-models.md

### [REJECT] reconstruct-state-enables-temporal-queries
Duplicate of existing belief `es-temporal-query-via-up-to`
- Source: entries/2026/05/29/topic-cqrs-read-models.md


### [ACCEPT] btree-wal-three-phase-commit
The B-tree WAL commit follows a strict three-phase protocol: (1) write WAL entries + fsync, (2) fsync data file via `page_manager.sync()`, (3) truncate WAL + fsync WAL fd — crash-safe at every interleaving because WAL entries are physical page images making replay idempotent.
- Source: entries/2026/05/29/topic-crash-recovery-invariants.md

### [ACCEPT] wal-batch-mode-loses-up-to-n-records
In WAL batch sync mode, up to `batch_sync_count - 1` records (default 99) can be lost on `kill -9` because `os.fsync()` is deferred until the batch threshold is reached; only `append_batch()` force-fsyncs regardless of mode.
- Source: entries/2026/05/29/topic-crash-recovery-invariants.md

### [ACCEPT] sstable-crash-leaves-zero-count-header
A crash between SSTableWriter creation and `finish()` leaves `entry_count=0` in the file header while data records exist on disk; `SSTableReader` trusts the header count and reads the file as empty, silently losing all written entries.
- Source: entries/2026/05/29/topic-crash-recovery-invariants.md

### [ACCEPT] event-store-memory-ahead-of-disk
Event store appends events to the in-memory `_events` list before calling `_persist_event`, with no rollback on write failure — a crash or I/O error leaves the in-memory state ahead of disk with no mechanism to reconcile.
- Source: entries/2026/05/29/topic-crash-recovery-invariants.md

### [ACCEPT] lsm-compact-no-atomic-rename
The LSM tree's `compact()` method does not use `os.rename` or `os.replace`; grep for atomic rename operations returns zero matches in the LSM module, meaning the compaction output is written directly to its final path with no atomic swap.
- Source: entries/2026/05/29/topic-crash-recovery-testing.md

### [ACCEPT] lsm-crash-test-ignores-compaction
`test_crash_recovery` in `test_lsm.py` only covers WAL replay for unflushed memtable entries; no test exercises crashes during SSTable compaction, leaving the most dangerous data-loss window (mid-compaction file deletion) untested.
- Source: entries/2026/05/29/topic-crash-recovery-testing.md

### [ACCEPT] compaction-crash-can-resurrect-deleted-keys
A crash during LSM compaction after tombstones are stripped from the merged output but before old SSTables are deleted can resurrect previously-deleted keys, because the tombstone that suppressed them no longer exists in any surviving SSTable.
- Source: entries/2026/05/29/topic-crash-recovery-testing.md

### [REJECT] wal-does-not-protect-flushed-data
Duplicates existing belief `lsm-wal-truncated-after-sstable-write` which already captures that the WAL is truncated after flush, leaving flushed SSTable data unprotected.
- Source: entries/2026/05/29/topic-crash-recovery-testing.md

### [REJECT] compaction-removes-tombstones
Duplicates existing beliefs `compact-purges-tombstones` and `lsm-compact-removes-tombstones-safely`.
- Source: entries/2026/05/29/topic-crash-recovery-testing.md

### [REJECT] lsht-crc-covers-key-and-value
Duplicates existing belief `all-three-implementations-use-payload-only-crc` which already states CRC covers data payloads (key+value), excluding header/framing metadata.
- Source: entries/2026/05/29/topic-crc-integrity-on-tombstones.md

### [REJECT] his-tombstone-is-zero-val-size
Duplicates existing belief `bitcask-tombstone-is-empty-string` which captures that hash-index-storage encodes deletion as an empty value, and `bitcask-tombstone-semantics` which covers the semantics.
- Source: entries/2026/05/29/topic-crc-integrity-on-tombstones.md

### [REJECT] his-no-integrity-check-on-read
Duplicates existing beliefs `hash-index-no-crc-by-design` and `silent-corruption-returns-garbage` which together capture that hash-index-storage performs no integrity verification on read.
- Source: entries/2026/05/29/topic-crc-integrity-on-tombstones.md

### [REJECT] lsht-scan-stops-on-first-crc-failure
Duplicates existing belief `corruption-terminates-scan` which states that a corrupted record terminates the current segment scan.
- Source: entries/2026/05/29/topic-crc-integrity-on-tombstones.md

### [REJECT] his-val-size-corruption-cascades
Duplicates existing belief `header-corruption-derails-sequential-scan` which states that corrupted key_size or val_size causes _scan_data_file to misparse all subsequent records.
- Source: entries/2026/05/29/topic-crc-integrity-on-tombstones.md

### [ACCEPT] zlib-crc32-is-iso3309
All 13 CRC call sites across the codebase use `zlib.crc32`, which implements the ISO 3309 polynomial (0xEDB88320); no module uses the Castagnoli polynomial (CRC-32C) that RocksDB and PostgreSQL prefer for hardware-accelerated checksumming.
- Source: entries/2026/05/29/topic-crc32-vs-crc32c.md

### [REJECT] lsm-has-no-checksumming
Duplicates existing belief `lsm-and-sstable-have-no-checksums` which already states neither module computes or verifies any checksums.
- Source: entries/2026/05/29/topic-crc32-vs-crc32c.md

### [ACCEPT] crc-mask-is-python2-artifact
The `& 0xFFFFFFFF` mask applied after `zlib.crc32` in all modules is a Python 2 compatibility artifact; Python 3's `zlib.crc32` already returns an unsigned 32-bit integer, making the mask harmless but unnecessary.
- Source: entries/2026/05/29/topic-crc32-vs-crc32c.md

### [ACCEPT] crc-polynomial-change-breaks-wire-format
Switching from ISO 3309 (`zlib.crc32`) to Castagnoli (CRC-32C) is a backward-incompatible wire format change; existing data files would fail CRC verification without a format version mechanism in the file header.
- Source: entries/2026/05/29/topic-crc32-vs-crc32c.md

### [REJECT] counting-bloom-counter-saturation-breaks-delete
Duplicates existing belief `cbf-saturated-counters-never-decrement` which already captures that saturated counters are never decremented during removal.
- Source: entries/2026/05/29/topic-cuckoo-filters.md

### [ACCEPT] counting-bloom-uses-byte-per-counter
CountingBloomFilter allocates one full byte per counter position (`bytearray(self._m)` at line 112) despite `counter_bits` defaulting to 4, using 8x the space of a standard BloomFilter's bit array rather than the expected 4x.
- Source: entries/2026/05/29/topic-cuckoo-filters.md

### [ACCEPT] no-cuckoo-or-xor-filter-in-repo
The repository has no cuckoo filter, quotient filter, or xor filter implementation; CountingBloomFilter is the only probabilistic membership structure supporting deletion.
- Source: entries/2026/05/29/topic-cuckoo-filters.md

### [ACCEPT] bloom-and-counting-share-hash-function
BloomFilter and CountingBloomFilter use the identical `_hashes()` double-hashing scheme (MD5-based, lines 9-14), producing the same bit/counter positions for identical parameters — so a BloomFilter and CountingBloomFilter with the same `m` and `k` will agree on membership queries for the same insertions.
- Source: entries/2026/05/29/topic-cuckoo-filters.md


### [ACCEPT] per-record-crc-cannot-detect-reordering
Independent per-record CRCs in `wal.py`, `btree.py`, and `bitcask.py` validate each record in isolation, so record deletion, reordering, or splicing produces a file where every CRC passes but the log is semantically corrupt
- Source: entries/2026/05/29/topic-cumulative-checksums.md

### [ACCEPT] truncate-plus-per-record-crc-is-dangerous-combination
The non-atomic `truncate()` in `wal.py` can produce reordered or incomplete files that pass per-record CRC validation; chained checksums would detect the corruption at the chain break point
- Source: entries/2026/05/29/topic-cumulative-checksums.md

### [ACCEPT] zlib-crc32-supports-chaining-natively
Python's `zlib.crc32(data, initial_value)` accepts an initial CRC value, meaning chained checksums could be implemented in this codebase by passing the previous frame's CRC as the seed with no additional hashing infrastructure
- Source: entries/2026/05/29/topic-cumulative-checksums.md

### [ACCEPT] chained-checksums-trade-random-access-for-log-integrity
Chained frame checksums (as in SQLite's WAL) prevent mid-log splicing, deletion, and reordering but require sequential validation from the chain start, eliminating the ability to validate or replay a single record in isolation
- Source: entries/2026/05/29/topic-cumulative-checksums.md

### [ACCEPT] broadcast-hash-join-requires-small-side-in-memory
`BroadcastHashJoin` loads the entire small dataset into a hash table at construction time via `_build_hash_table`, so the small side must fit in a single mapper's memory
- Source: entries/2026/05/29/topic-ddia-ch10-reduce-side-joins.md

### [ACCEPT] partitioned-hash-join-partitions-both-sides
`PartitionedHashJoin.join()` calls `partition_dataset()` on both inputs before joining, meaning partitions are computed at join time rather than assumed from storage layout
- Source: entries/2026/05/29/topic-ddia-ch10-reduce-side-joins.md

### [REJECT] mapreduce-shuffle-uses-hash-partitioning
Duplicates existing belief `mr-hash-partitions-keys`
- Source: entries/2026/05/29/topic-ddia-ch10-reduce-side-joins.md

### [ACCEPT] map-side-joins-track-mapper-id-on-output
All three map-side join strategies attach a `_mapper_id` field to each output record, enabling verification that partitions/mappers operated independently
- Source: entries/2026/05/29/topic-ddia-ch10-reduce-side-joins.md

### [REJECT] reduce-side-sort-group-uses-itertools-groupby
Duplicates existing belief `mr-groupby-requires-presort` which already covers the itertools.groupby dependency on pre-sorted input
- Source: entries/2026/05/29/topic-ddia-ch10-reduce-side-joins.md

### [ACCEPT] wal-is-source-of-truth
The unbundled database's `StorageEngine` is fully derivable from the `WriteAheadLog`; calling `rebuild()` replays the entire WAL and reproduces identical state, making the log the authoritative record
- Source: entries/2026/05/29/topic-ddia-ch12-unbundling.md

### [REJECT] cdc-old-value-required-for-consistency
Duplicates the combination of existing beliefs `unbundled-db-cdc-events-carry-old-value` and `secondary-index-stale-entry-cleanup`
- Source: entries/2026/05/29/topic-ddia-ch12-unbundling.md

### [ACCEPT] derived-systems-are-position-tracked
Every `DerivedSystem` in the unbundled database tracks a `position` cursor indicating how far through the CDC log it has consumed, enabling independent catch-up and ensuring consumers can resume from where they left off
- Source: entries/2026/05/29/topic-ddia-ch12-unbundling.md

### [ACCEPT] cdc-log-compact-keeps-latest-per-key
`CDCLog.compact()` in `change-data-capture/cdc.py` retains only the latest event per `(table, key)`, bounding log size while allowing new consumers to bootstrap current state from the compacted log
- Source: entries/2026/05/29/topic-ddia-ch12-unbundling.md

### [ACCEPT] event-store-optimistic-concurrency
`EventStore.append()` accepts an `expected_version` parameter and rejects appends when the stream's current version doesn't match, implementing optimistic concurrency control for event streams
- Source: entries/2026/05/29/topic-ddia-ch12-unbundling.md

### [ACCEPT] fencing-token-counter-is-global
The `LockService._counter` is a single monotonically increasing integer shared across all lock names, not per-lock, ensuring tokens from different locks are totally ordered
- Source: entries/2026/05/29/topic-ddia-ch8-process-pauses.md

### [ACCEPT] fenced-server-rejects-strictly-lower-tokens
`FencedResourceServer.write` rejects writes where `fencing_token < highest` (strict less-than, not less-than-or-equal), meaning same-token retries succeed — enabling idempotent writes with the same lock acquisition
- Source: entries/2026/05/29/topic-ddia-ch8-process-pauses.md

### [ACCEPT] lock-expiry-enables-reacquisition-without-release
`LockService.acquire` checks `is_expired(current_time)` before rejecting a competing acquire, so a crashed or GC-paused client's lock is automatically reclaimable after TTL without requiring explicit release
- Source: entries/2026/05/29/topic-ddia-ch8-process-pauses.md

### [ACCEPT] fencing-tokens-do-not-expire-at-resource-server
The `FencedResourceServer` has no TTL or expiration logic for tokens — once a token number is seen, all lower tokens are permanently rejected for that resource
- Source: entries/2026/05/29/topic-ddia-ch8-process-pauses.md

### [ACCEPT] client-holds-stale-token-after-lock-expiry
The `Client._held_tokens` dict is never cleared on lock expiry — the client retains and may use a token whose corresponding lock has already been acquired by another client
- Source: entries/2026/05/29/topic-ddia-ch8-process-pauses.md

### [REJECT] count-sort-are-materialization-barriers
Count part duplicates existing belief `count-is-barrier`; Sort's barrier nature is implicit in `sort-external-merge`
- Source: entries/2026/05/29/topic-ddia-chapter-10-batch-processing.md

### [REJECT] sort-merge-join-cartesian-on-duplicates
Duplicates existing belief `map-side-join-cartesian-on-duplicates`
- Source: entries/2026/05/29/topic-ddia-chapter-10-batch-processing.md

### [ACCEPT] mapreduce-materializes-at-phase-boundaries
MapReduce in `mapreduce.py` writes intermediate data to JSON partition files between map and reduce phases, creating a full materialization barrier that prevents streaming from mapper to reducer
- Source: entries/2026/05/29/topic-ddia-chapter-10-batch-processing.md

### [ACCEPT] stream-join-bounds-materialization-via-watermarks
`StreamJoinProcessor` expires buffered events when they fall below `watermark - window_duration`, bounding memory at the cost of potentially dropping late-arriving matches
- Source: entries/2026/05/29/topic-ddia-chapter-10-batch-processing.md

### [ACCEPT] unbundled-db-composes-via-log
The unbundled database wires independent subsystems (WAL, storage engine, CDC, derived systems) through a shared append-only log with independent consumer positions, applying Unix-style composition at the system architecture level
- Source: entries/2026/05/29/topic-ddia-chapter-10-batch-processing.md

### [ACCEPT] pipeline-no-inter-stage-schema-validation
Pipeline stages pass untyped tuples with no validation that the output shape of one stage matches the input expectations of the next; a mismatched stage composition fails at runtime, not at construction time
- Source: entries/2026/05/29/topic-ddia-chapter-10-batch-processing.md


### [ACCEPT] range-partition-split-at-median
`Partition.split` always divides at the median key (index `len // 2`), which can produce uneven partitions when key distribution is skewed
- Source: entries/2026/05/29/topic-ddia-chapter-6-partitioning.md

### [ACCEPT] consistent-hash-ring-uses-md5
All consistent hash ring positions are computed via MD5 truncated to 32 bits; the hash function is not pluggable
- Source: entries/2026/05/29/topic-ddia-chapter-6-partitioning.md

### [REJECT] document-partitioned-query-touches-all
Duplicate of existing `doc-partitioned-query-touches-all`
- Source: entries/2026/05/29/topic-ddia-chapter-6-partitioning.md

### [REJECT] term-partitioned-write-fans-out
Duplicate of existing `term-partitioned-write-touches-multiple`
- Source: entries/2026/05/29/topic-ddia-chapter-6-partitioning.md

### [REJECT] range-partition-routing-is-log-n
Duplicate of existing `range-partitioning-boundary-routing-bisect`
- Source: entries/2026/05/29/topic-ddia-chapter-6-partitioning.md

### [ACCEPT] consistent-hash-default-150-vnodes
`ConsistentHashRing` defaults to 150 virtual nodes per physical node, and exposes `load_imbalance` (max_load / avg_load) to measure distribution quality
- Source: entries/2026/05/29/topic-ddia-chapter-6-partitioning.md

### [ACCEPT] consistent-hash-weight-scales-vnodes
The `weight` parameter on `add_node` scales virtual node count proportionally, supporting heterogeneous nodes with different capacities
- Source: entries/2026/05/29/topic-ddia-chapter-6-partitioning.md

### [ACCEPT] producer-keyed-hash-unkeyed-roundrobin
Partitioned log `Producer` hashes keyed messages to a fixed partition for ordering guarantees, and round-robins unkeyed messages across partitions for load distribution
- Source: entries/2026/05/29/topic-ddia-chapter-6-partitioning.md

### [ACCEPT] wal-is-source-of-truth
`StorageEngine` has no direct mutation API; all state changes flow through `WriteAheadLog.append` then `StorageEngine.apply`, and `rebuild(wal)` reproduces identical state from the log alone — making the storage engine derived data
- Source: entries/2026/05/29/topic-ddia-unbundling-concept.md

### [ACCEPT] cdc-old-value-required-for-index-consistency
`SecondaryIndex.process_event` depends on `CDCEvent.old_value` to remove stale index entries during updates; without before-images, incremental index maintenance produces phantom references
- Source: entries/2026/05/29/topic-ddia-unbundling-concept.md

### [ACCEPT] derived-systems-independently-position-tracked
Each `DerivedSystem` tracks its own LSN `position` independently, allowing consumers to fall behind or catch up at different rates without coordinator state or blocking
- Source: entries/2026/05/29/topic-ddia-unbundling-concept.md

### [REJECT] lazy-propagation-via-explicit-flush
Duplicate of existing `unbundled-db-flush-required-for-derived`
- Source: entries/2026/05/29/topic-ddia-unbundling-concept.md

### [ACCEPT] event-sourcing-and-cdc-converge-on-projection-pattern
Both `Projection.catch_up` and `DerivedSystem.process_event` track a position cursor, pull/receive ordered events, and apply type-dispatched handlers — the same structural pattern solving "derived data" from different starting points
- Source: entries/2026/05/29/topic-ddia-unbundling-concept.md

### [ACCEPT] cdc-determines-insert-vs-update-by-old-value
The CDC layer distinguishes `insert` from `update` events by checking whether `old_value is None`; the WAL only knows `PUT` and `DELETE`, so the semantic enrichment happens at the CDC boundary
- Source: entries/2026/05/29/topic-ddia-unbundling-concept.md

### [ACCEPT] event-store-optimistic-concurrency-via-expected-version
`EventStore.append` takes an `expected_version` parameter for optimistic concurrency on stream appends, rejecting writes when the stream has advanced past the caller's version
- Source: entries/2026/05/29/topic-ddia-unbundling-concept.md

### [ACCEPT] crdts-are-state-based-cvrdts
All four CRDT types (GCounter, PNCounter, LWWRegister, ORSet) use state-based replication via a `merge()` method that implements a join-semilattice; no operation log or causal delivery is used
- Source: entries/2026/05/29/topic-delta-state-crdts.md

### [ACCEPT] orset-tombstones-grow-monotonically
`ORSet._tombstones` is only ever unioned during `merge()` and added to during `remove()`; there is no compaction or garbage collection, so the tombstone set grows without bound
- Source: entries/2026/05/29/topic-delta-state-crdts.md

### [ACCEPT] merge-mutates-self-and-returns-self
Every CRDT `merge()` method mutates the receiver in place and returns `self`, enabling chaining but meaning callers must `deepcopy()` before merging if the original state must be preserved
- Source: entries/2026/05/29/topic-delta-state-crdts.md

### [ACCEPT] lww-register-tiebreaks-on-replica-id
When two `LWWRegister` writes have identical timestamps, the one with the lexicographically higher `replica_id` wins, making conflict resolution deterministic but arbitrary
- Source: entries/2026/05/29/topic-delta-state-crdts.md

### [REJECT] wal-rotation-missing-dir-fsync
Duplicate of existing `wal-rotation-creates-unsynced-directory-entry` and `wal-no-directory-fsync`
- Source: entries/2026/05/29/topic-directory-fsync-gap.md

### [REJECT] sstable-writer-no-fsync
Duplicate of existing `sstable-writer-no-fsync-on-close`
- Source: entries/2026/05/29/topic-directory-fsync-gap.md

### [REJECT] lsm-wal-truncate-before-sstable-durable
Duplicate of existing `lsm-flush-double-loss-window`
- Source: entries/2026/05/29/topic-directory-fsync-gap.md

### [REJECT] no-dir-fsync-anywhere
Duplicate of existing `all-fsync-sites-data-integrity-only` (all fsync calls target data file descriptors only, never directories)
- Source: entries/2026/05/29/topic-directory-fsync-gap.md

### [REJECT] wal-do-sync-false-durability
Covered by combination of existing `wal-rotation-creates-unsynced-directory-entry` and `rotation-always-fsyncs`
- Source: entries/2026/05/29/topic-directory-fsync-gap.md

### [REJECT] wal-rotation-creates-unsynced-directory-entry
Already exists as an accepted belief with this exact ID
- Source: entries/2026/05/29/topic-directory-fsync.md

### [REJECT] lsm-sstable-flush-has-no-fsync
Covered by existing `fsync-used-only-for-appends` (fsync never used for SSTable creation) and `lsm-compaction-deletes-without-fsync-barrier`
- Source: entries/2026/05/29/topic-directory-fsync.md

### [REJECT] recovery-assumes-directory-entries-durable
Duplicate of existing `wal-recovery-depends-on-listdir`
- Source: entries/2026/05/29/topic-directory-fsync.md

### [REJECT] bitcask-rename-without-dir-fsync
Duplicate of existing `rename-without-dir-barrier`
- Source: entries/2026/05/29/topic-directory-fsync.md


### [ACCEPT] gossip-carries-membership-not-data
`GossipNode.receive_gossip` merges heartbeat counters and node status (alive/suspected/dead) only; it never exchanges application key-value data, making it a SWIM-style failure detector rather than a data replication protocol
- Source: entries/2026/05/29/topic-dynamo-anti-entropy.md

### [ACCEPT] receive-replica-is-passive-endpoint
`_receive_replica` in `vector-clocks/vector_clock.py` is the receiving side of anti-entropy replication; nothing in the codebase calls it automatically — tests simulate the call manually, and no gossip or sync module invokes it
- Source: entries/2026/05/29/topic-dynamo-anti-entropy.md

### [ACCEPT] anti-entropy-layers-are-decoupled
The gossip, merkle-tree, vector-clock, read-repair, and hinted-handoff modules are independent implementations with no cross-imports; composing them into a full Dynamo-style anti-entropy pipeline is left to the integrator
- Source: entries/2026/05/29/topic-dynamo-anti-entropy.md

### [ACCEPT] merkle-diff-enables-efficient-sync
`MerkleTree.diff` compares two trees in O(log n) by short-circuiting at matching subtree hashes, providing the divergence-detection layer that would feed `_receive_replica` with only changed keys rather than a full key scan
- Source: entries/2026/05/29/topic-dynamo-anti-entropy.md

### [ACCEPT] gossip-failure-detection-gates-replication
`GossipNode.get_alive_members` returns the liveness set that anti-entropy, read-repair, and hinted-handoff all depend on to decide which nodes to contact — gossip provides the membership layer that every data-exchange layer needs
- Source: entries/2026/05/29/topic-dynamo-anti-entropy.md

### [REJECT] ets-is-zero-copy-across-processes
General Erlang/BEAM VM fact, not a testable claim about this codebase
- Source: entries/2026/05/29/topic-erlang-ets-concurrency-model.md

### [REJECT] gil-does-not-cross-process-boundaries
General Python runtime fact, not a codebase-specific architectural claim
- Source: entries/2026/05/29/topic-erlang-ets-concurrency-model.md

### [ACCEPT] python-bitcask-no-keydir-sharing
Each `BitcaskStore` instance independently rebuilds the full in-memory index from disk; there is no shared-memory mechanism to preserve or share the keydir across instances even within the same process, unlike Erlang's ETS-backed keydir
- Source: entries/2026/05/29/topic-erlang-ets-concurrency-model.md

### [REJECT] ets-insert-is-per-key-atomic
Erlang ETS implementation detail, not a claim about this Python codebase
- Source: entries/2026/05/29/topic-erlang-ets-concurrency-model.md

### [ACCEPT] append-batch-version-check-is-precondition
`append_batch` checks `expected_version` once before the write loop; it does not re-check after each event, so the guard is a pre-condition gate that rejects the entire batch cleanly on mismatch, not a per-event invariant
- Source: entries/2026/05/29/topic-event-sourcing-concurrency-model.md

### [ACCEPT] append-batch-no-rollback
If an exception occurs mid-batch in `append_batch` (e.g., disk I/O failure in `_persist_event`), already-appended events remain in `_events`, `_streams`, and on disk with no rollback — the process continues with inconsistent in-memory state
- Source: entries/2026/05/29/topic-event-sourcing-concurrency-model.md

### [REJECT] append-batch-defers-subscriber-notification
Duplicates existing belief `batch-notifies-after-all-stored`
- Source: entries/2026/05/29/topic-event-sourcing-concurrency-model.md

### [ACCEPT] event-store-single-writer-assumption
The event store's `expected_version` check-then-act pattern in `append` and `append_batch` has no locking, making the optimistic concurrency guard safe only under a single-writer concurrency model — concurrent writers create a TOCTOU race
- Source: entries/2026/05/29/topic-event-sourcing-concurrency-model.md

### [REJECT] snapshot-captures-state-and-position
Duplicates existing belief `es-snapshot-saves-state-and-position`
- Source: entries/2026/05/29/topic-event-sourcing-snapshots.md

### [ACCEPT] catch-up-uses-position-plus-one
`Projection.catch_up()` calls `read_all(from_position=self._position + 1)`, so loading a snapshot sets the exact boundary — only events after the snapshot's position are replayed, making reconstruction cost proportional to events-since-snapshot
- Source: entries/2026/05/29/topic-event-sourcing-snapshots.md

### [ACCEPT] deepcopy-prevents-snapshot-corruption
`save_snapshot()` uses `copy.deepcopy` on the projection state dict to create an independent copy, ensuring subsequent event processing during `catch_up()` does not mutate the saved snapshot
- Source: entries/2026/05/29/topic-event-sourcing-snapshots.md

### [REJECT] wal-py-fsync-on-every-sync-write
Covered by existing beliefs `batch-mode-defers-fsync` and `wal-do-sync-branches-mutually-exclusive`
- Source: entries/2026/05/29/topic-fdatasync-vs-fsync-optimization.md

### [REJECT] lsm-wal-missing-fsync
Duplicates existing belief `lsm-wal-has-no-fsync`
- Source: entries/2026/05/29/topic-fdatasync-vs-fsync-optimization.md

### [ACCEPT] fdatasync-safe-for-all-append-paths
Every file opened in append mode (`"ab"`) in this codebase writes sequentially with monotonically increasing file size, making `fdatasync` a safe drop-in replacement for `fsync` on those paths (WAL, Bitcask data files, B-tree WAL)
- Source: entries/2026/05/29/topic-fdatasync-vs-fsync-optimization.md

### [ACCEPT] fdatasync-not-available-on-darwin
Python's `os.fdatasync` exists only on Linux, not macOS/Darwin; any switch from `os.fsync` requires a platform fallback via `getattr(os, 'fdatasync', os.fsync)` to work on the current development environment
- Source: entries/2026/05/29/topic-fdatasync-vs-fsync-optimization.md


### [ACCEPT] free-list-is-lifo-stack
The B-tree's free page list is a LIFO stack: `free_page` pushes onto the head and `allocate_page` pops from the head, so pages are reused in reverse order of when they were freed.
- Source: entries/2026/05/29/topic-free-list-fragmentation.md

### [ACCEPT] btree-file-never-shrinks
The B-tree data file only grows (via `next_free` bump in `allocate_page`) and never shrinks; no truncate, compact, or defrag operation exists anywhere in the module.
- Source: entries/2026/05/29/topic-free-list-fragmentation.md

### [ACCEPT] free-page-stores-only-seven-bytes
Each free-list page occupies a full page (default 4096 bytes) but stores only 7 bytes of useful data (3-byte zero header + 4-byte next pointer); the remaining bytes are zero padding from `write_page`.
- Source: entries/2026/05/29/topic-free-list-fragmentation.md

### [ACCEPT] free-list-lifo-decorrelates-layout
LIFO free-list reuse means delete-then-reinsert cycles progressively decorrelate logical key order from physical page order, defeating sequential I/O readahead for range scans.
- Source: entries/2026/05/29/topic-free-list-fragmentation.md

### [REJECT] free-list-preferred-over-growth
Duplicates existing belief `allocate-page-prefers-free-list`.
- Source: entries/2026/05/29/topic-free-list-fragmentation.md

### [ACCEPT] wal-checkpoint-forces-fsync
`checkpoint()` calls `_do_sync(force=True)`, making the checkpoint marker durable on disk regardless of the configured sync mode, so recovery boundaries are never lost even in `"none"` or `"batch"` mode.
- Source: entries/2026/05/29/topic-fsync-durability-tradeoffs.md

### [REJECT] wal-batch-and-checkpoint-always-fsync
The batch aspect duplicates existing belief `wal-batch-always-fsyncs`; checkpoint aspect captured separately.
- Source: entries/2026/05/29/topic-fsync-durability-tradeoffs.md

### [REJECT] wal-none-mode-skips-all-sync
Duplicates existing belief `none-mode-never-syncs-on-append`.
- Source: entries/2026/05/29/topic-fsync-durability-tradeoffs.md

### [REJECT] wal-batch-mode-counter-resets-on-sync
Duplicates existing belief `batch-mode-defers-fsync` which covers the batch sync mechanism.
- Source: entries/2026/05/29/topic-fsync-durability-tradeoffs.md

### [REJECT] wal-rotation-forces-fsync
Duplicates existing beliefs `rotation-always-fsyncs` and `wal-rotate-fsync-before-close`.
- Source: entries/2026/05/29/topic-fsync-durability-tradeoffs.md

### [ACCEPT] btree-single-file-avoids-dir-fsync-gap
The B-tree's PageManager writes to a single pre-existing data file opened at construction, so normal operations never create new files and avoid the directory fsync gap that affects segment-based engines during rotation and compaction.
- Source: entries/2026/05/29/topic-fsync-guarantees.md

### [REJECT] log-structured-no-fsync
Duplicates existing belief `log-structured-bitcask-no-fsync`.
- Source: entries/2026/05/29/topic-fsync-guarantees.md

### [REJECT] hash-index-conditional-fsync
Duplicates existing belief `hash-index-sync-writes-controls-fsync`.
- Source: entries/2026/05/29/topic-fsync-guarantees.md

### [REJECT] lsm-wal-flush-only
Duplicates existing belief `lsm-wal-has-no-fsync`.
- Source: entries/2026/05/29/topic-fsync-guarantees.md

### [REJECT] standalone-wal-proper-sync
Claim that standalone WAL pairs flush+fsync "in all three sync modes" is inaccurate — `"none"` mode intentionally skips fsync, contradicting the entry's own analysis.
- Source: entries/2026/05/29/topic-fsync-guarantees.md

### [REJECT] flush-fsync-pairing-inconsistent
Derivable from the combination of existing beliefs `lsm-wal-has-no-fsync` and `wal-fsync-per-entry`.
- Source: entries/2026/05/29/topic-fsync-guarantees.md

### [REJECT] wal-rotation-creates-new-files
The new-file-creation behavior during rotation is already implied by existing rotation beliefs; the primary significance is about ext4 `auto_da_alloc` interaction, which is filesystem behavior rather than a codebase invariant.
- Source: entries/2026/05/29/topic-fsync-on-ext4-vs-other-filesystems.md

### [REJECT] bitcask-rename-is-not-replace
Already covered by `delete-before-rename-ordering` and `rename-without-dir-barrier`; the ext4 `auto_da_alloc` interaction is filesystem knowledge, not a codebase claim.
- Source: entries/2026/05/29/topic-fsync-on-ext4-vs-other-filesystems.md


### [ACCEPT] flush-before-fsync-invariant
Every `os.fsync()` call in the codebase is immediately preceded by a `.flush()` call, ensuring Python's userspace buffer is drained to the kernel page cache before requesting stable storage persistence.
- Source: entries/2026/05/29/topic-fsync-vs-flush-semantics.md

### [ACCEPT] no-cross-file-fsync-ordering-protocol
The standalone WAL module provides no mechanism or documentation for enforcing the required WAL-before-data fsync ordering; callers must implement the `fsync(wal)` → `write(data)` → `fsync(data)` protocol themselves.
- Source: entries/2026/05/29/topic-fsync-ordering-guarantees.md

### [ACCEPT] macos-fsync-not-durable-without-fullfsync
On macOS (the development platform), `os.fsync()` may not flush the disk's hardware write cache; true power-loss durability requires `fcntl(fd, F_FULLFSYNC)` which is never used in the codebase.
- Source: entries/2026/05/29/topic-fsync-ordering-guarantees.md

### [ACCEPT] btree-double-fsync-per-mutation
B-tree mutations pay for `os.fsync()` twice: once when writing the WAL entry (`btree.py:137`) and again when committing the page to the data file (`btree.py:105`), with the WAL truncated only after the data file sync confirms durability.
- Source: entries/2026/05/29/topic-fsync-vs-fdatasync.md

### [ACCEPT] wal-no-concurrent-group-commit
The WAL uses a single `threading.Lock` and a write counter for batch sync rather than a concurrent waiter queue, so it cannot amortize fsync cost across concurrent callers the way PostgreSQL's group commit does.
- Source: entries/2026/05/29/topic-group-commit-optimization.md

### [REJECT] lsm-wal-no-fsync
Duplicate of existing belief `lsm-wal-has-no-fsync`.
- Source: entries/2026/05/29/topic-fsync-ordering-guarantees.md

### [REJECT] wal-sync-mode-batch-window
Duplicate of existing belief `wal-batch-sync-count-controls-durability-window`.
- Source: entries/2026/05/29/topic-fsync-ordering-guarantees.md

### [REJECT] fsync-only-durability
Covered by existing beliefs `fdatasync-never-used` and `no-o-direct-usage`.
- Source: entries/2026/05/29/topic-fsync-vs-fdatasync-vs-o-direct.md

### [REJECT] lsm-wal-missing-fsync
Duplicate of existing belief `lsm-wal-has-no-fsync`.
- Source: entries/2026/05/29/topic-fsync-vs-fdatasync-vs-o-direct.md

### [REJECT] bitcask-sync-writes-flag
Duplicate of existing belief `hash-index-sync-writes-controls-fsync`.
- Source: entries/2026/05/29/topic-fsync-vs-fdatasync-vs-o-direct.md

### [REJECT] wal-batch-sync-deferred-durability
Duplicate of existing belief `batch-mode-defers-fsync`.
- Source: entries/2026/05/29/topic-fsync-vs-fdatasync-vs-o-direct.md

### [REJECT] fsync-exclusive-no-fdatasync
Duplicate of existing belief `fdatasync-never-used`.
- Source: entries/2026/05/29/topic-fsync-vs-fdatasync.md

### [REJECT] wal-sync-mode-controls-durability-window
Duplicate of existing belief `wal-batch-sync-count-controls-durability-window`.
- Source: entries/2026/05/29/topic-fsync-vs-fdatasync.md

### [REJECT] lsm-wal-flush-without-fsync
Duplicate of existing belief `lsm-wal-has-no-fsync`.
- Source: entries/2026/05/29/topic-fsync-vs-fdatasync.md

### [REJECT] bitcask-per-record-fsync-default
Duplicate of existing belief `bitcask-fsync-per-record-default`.
- Source: entries/2026/05/29/topic-fsync-vs-fdatasync.md

### [REJECT] btree-wal-sync-every-write
Duplicate of existing belief `btree-wal-fsync-per-entry`.
- Source: entries/2026/05/29/topic-fsync-vs-flush-semantics.md

### [REJECT] no-fdatasync-usage
Duplicate of existing belief `fdatasync-never-used`.
- Source: entries/2026/05/29/topic-fsync-vs-flush-semantics.md

### [REJECT] wal-sync-mode-configurable
The sync mode configurability is covered by `wal-batch-sync-count-controls-durability-window`; the force-sync behavior of `append_batch` and `checkpoint` is covered by `wal-batch-always-fsyncs` and `wal-checkpoint-forces-sync`.
- Source: entries/2026/05/29/topic-fsync-vs-flush-semantics.md

### [REJECT] wal-batch-mode-loses-up-to-N-writes
Duplicate of existing belief `wal-batch-sync-count-controls-durability-window`.
- Source: entries/2026/05/29/topic-group-commit-optimization.md

### [REJECT] append-batch-always-fsyncs
Duplicate of existing belief `wal-batch-always-fsyncs`.
- Source: entries/2026/05/29/topic-group-commit-optimization.md

### [REJECT] lsm-wal-never-fsyncs
Duplicate of existing belief `lsm-wal-has-no-fsync`.
- Source: entries/2026/05/29/topic-group-commit-optimization.md


## Entry: `topic-hash-partitioning-skew.md`

### [ACCEPT] hash-mod-partition-count-is-static
In `mapreduce.py`, `num_reducers` is fixed at job creation (`num_reducers: int = 2`) and never changes during execution; all partition assignments are determined by `hash(k) % num_reducers` with no dynamic rebalancing
- Source: entries/2026/05/29/topic-hash-partitioning-skew.md

### [REJECT] range-partition-splits-at-median
Duplicates existing belief `range-partitioning-split-at-median-index`
- Source: entries/2026/05/29/topic-hash-partitioning-skew.md

### [REJECT] range-partition-boundaries-are-contiguous
Duplicates existing belief `range-partitioning-contiguous-half-open-coverage`
- Source: entries/2026/05/29/topic-hash-partitioning-skew.md

### [ACCEPT] hash-mod-destroys-key-order
Hash-mod partitioning (`hash(k) % num_reducers`) scatters lexicographically adjacent keys across different partitions, making range queries require a full scatter-gather across all reducers; MapReduce `run()` re-sorts the final results to compensate
- Source: entries/2026/05/29/topic-hash-partitioning-skew.md

### [ACCEPT] range-partition-merge-is-greedy-left-to-right
`merge_small_partitions` sweeps left-to-right and greedily merges adjacent pairs whose combined size is at or below `min_partition_size`, which can produce different results than an optimal merge strategy
- Source: entries/2026/05/29/topic-hash-partitioning-skew.md

### [ACCEPT] hash-mod-skew-shared-across-modules
Three modules use the same `hash(k) % num_partitions` pattern with fixed partition counts and no hot-key mitigation: `mapreduce.py:109`, `secondary_index_partitioning.py:56`, and `partitioned_log.py:149` — all share the straggler vulnerability where a single hot key overwhelms one partition
- Source: entries/2026/05/29/topic-hash-partitioning-skew.md

### [ACCEPT] range-partition-sequential-insert-skew
Range partitioning has its own skew risk: keys arriving in sorted order all hit the rightmost partition; `Partition.split()` mitigates this for data already stored by splitting at the median, but cannot prevent temporary hotspots during sequential ingestion
- Source: entries/2026/05/29/topic-hash-partitioning-skew.md

---

## Entry: `topic-hint-file-effectiveness.md`

### [REJECT] hint-file-omits-values
Duplicates existing beliefs `hash-index-hint-self-sufficient` and `hint-format-diverges-between-implementations`
- Source: entries/2026/05/29/topic-hint-file-effectiveness.md

### [REJECT] scan-reads-then-discards-values
Duplicates existing belief `data-scan-skips-value-bytes`
- Source: entries/2026/05/29/topic-hint-file-effectiveness.md

### [REJECT] hint-files-only-after-compaction
Duplicates existing belief `hint-written-only-during-compaction`
- Source: entries/2026/05/29/topic-hint-file-effectiveness.md

### [REJECT] rebuild-prefers-hint-over-scan
Duplicates existing belief `bitcask-hint-files-skip-scan`
- Source: entries/2026/05/29/topic-hint-file-effectiveness.md

### [REJECT] two-bitcask-variants-differ-in-hint-format
Duplicates existing belief `hint-format-diverges-between-implementations`
- Source: entries/2026/05/29/topic-hint-file-effectiveness.md

---

## Entry: `topic-hint-file-integrity.md`

### [REJECT] missing-hint-safe-fallback
Duplicates existing beliefs `bitcask-hint-files-skip-scan` and `hint-corrupt-worse-than-missing` (which explicitly states a missing hint triggers a CRC-checked full scan)
- Source: entries/2026/05/29/topic-hint-file-integrity.md

### [REJECT] log-structured-crc-bypassed-by-hints
Duplicates existing beliefs `bitcask-crc-validated-on-read-only` and `hint-corrupt-worse-than-missing`, which together establish that hint loading skips CRC verification
- Source: entries/2026/05/29/topic-hint-file-integrity.md

### [REJECT] subtle-corruption-deferred-detection
Covered by existing beliefs `hint-corrupt-worse-than-missing` (silent bad offsets) and `bitcask-crc32-raises-corruption-error` (CRC mismatch at read time)
- Source: entries/2026/05/29/topic-hint-file-integrity.md

### [REJECT] hash-index-assert-partial-guard
Duplicates existing belief `bitcask-no-checksum-validation` which states "the only corruption guard is the `assert read_key == key` in `get()`"
- Source: entries/2026/05/29/topic-hint-file-integrity.md

---

## Entry: `topic-hint-file-tombstone-encoding.md`

### [REJECT] hash-index-hint-file-ignores-tombstones
Duplicates existing belief `bitcask-tombstone-handling-asymmetry` which states `_load_hint_file` unconditionally inserts entries and relies on compaction having already stripped tombstones
- Source: entries/2026/05/29/topic-hint-file-tombstone-encoding.md

### [REJECT] hash-index-data-scan-detects-tombstones
Duplicates existing belief `bitcask-tombstone-is-empty-string` which covers both `_scan_data_file` checking `val_size == 0` and `get` checking `value == ""`
- Source: entries/2026/05/29/topic-hint-file-tombstone-encoding.md

### [REJECT] log-structured-hint-format-lacks-value-size
Duplicates existing belief `lsht-hint-no-tombstone-flag` which states the `!II` format has no field to distinguish live records from tombstones
- Source: entries/2026/05/29/topic-hint-file-tombstone-encoding.md

### [REJECT] two-tombstone-strategies-diverge
Covered by existing beliefs `bitcask-tombstone-is-empty-string` (hash-index uses `value_size == 0`) and `log-structured-sentinel-is-in-band` (log-structured uses sentinel byte string)
- Source: entries/2026/05/29/topic-hint-file-tombstone-encoding.md

---

## Entry: `topic-hint-file-tombstone-handling.md`

### [REJECT] hint-format-lacks-tombstone-flag
Duplicates existing beliefs `bitcask-tombstone-handling-asymmetry` and `lsht-hint-no-tombstone-flag`
- Source: entries/2026/05/29/topic-hint-file-tombstone-handling.md

### [REJECT] load-hint-unconditional-keydir-insert
Duplicates existing belief `bitcask-tombstone-handling-asymmetry` which explicitly describes `_load_hint_file` unconditionally inserting entries
- Source: entries/2026/05/29/topic-hint-file-tombstone-handling.md

### [REJECT] tombstone-sentinel-is-empty-string
Duplicates existing belief `bitcask-tombstone-is-empty-string`
- Source: entries/2026/05/29/topic-hint-file-tombstone-handling.md

### [REJECT] hint-file-compaction-resurrection
Covered by the combination of existing beliefs `bitcask-tombstone-handling-asymmetry` (hint files don't check tombstones), `bitcask-hint-files-exclude-tombstones` (hint-based recovery can't distinguish deleted from never-existed), and `tombstone-removal-requires-full-key-coverage` (incomplete compaction can resurrect deleted keys)
- Source: entries/2026/05/29/topic-hint-file-tombstone-handling.md


### [ACCEPT] wal-no-begin-marker
The WAL protocol has no BEGIN record type, making it impossible during replay to distinguish a standalone PUT from the first record of a multi-record batch — the structural root cause of why COMMIT markers cannot enforce atomicity.
- Source: entries/2026/05/29/topic-incomplete-batch-recovery.md

### [REJECT] wal-replay-includes-uncommitted-puts
Duplicates `replay-does-not-enforce-batch-atomicity` and `wal-replay-ignores-commit`, which already establish that replay returns all PUT/DELETE records regardless of COMMIT presence.
- Source: entries/2026/05/29/topic-incomplete-batch-recovery.md

### [REJECT] wal-append-batch-single-write
Duplicates `wal-batch-single-write` which already captures that `append_batch()` serializes all records plus COMMIT into one `write()` call.
- Source: entries/2026/05/29/topic-incomplete-batch-recovery.md

### [REJECT] wal-crash-recovery-test-gap
Test coverage observation, not an architectural invariant — someone adding the test would invalidate this belief immediately.
- Source: entries/2026/05/29/topic-incomplete-batch-recovery.md

### [REJECT] wal-read-record-truncation-safe
Duplicates `wal-partial-read-is-eof` which already establishes that short reads are treated as EOF.
- Source: entries/2026/05/29/topic-incomplete-batch-recovery.md

### [ACCEPT] btree-uses-bisect-not-linear-scan
In-node key lookup uses `bisect_left`/`bisect_right` (binary search), making within-node search O(log B) rather than the textbook O(B) linear scan — shifting the optimal branching factor higher than classical B-tree analysis predicts.
- Source: entries/2026/05/29/topic-index-size-vs-scan-cost-benchmarking.md

### [REJECT] no-block-size-parameter-exists
Trivial API naming fact — the branching factor parameter name is already evident from any test or constructor call.
- Source: entries/2026/05/29/topic-index-size-vs-scan-cost-benchmarking.md

### [REJECT] page-io-counters-are-resettable
Duplicates `btree-counters-reset-per-operation` and `iter-no-counter-reset` which already cover the counter reset behavior.
- Source: entries/2026/05/29/topic-index-size-vs-scan-cost-benchmarking.md

### [REJECT] page-size-constrains-max-keys
Contradicts `no-page-size-enforcement-in-serialize` — serialization does not enforce page size limits, so the constraint is not actively enforced in code.
- Source: entries/2026/05/29/topic-index-size-vs-scan-cost-benchmarking.md

### [ACCEPT] internal-nodes-no-sibling-pointer
Internal nodes have no sibling pointer field; their binary layout is `[type:1B][num_keys:2B][child₀:4B]([keylen:2B][key][child:4B])...` with no right-link, and `_deserialize_internal` returns a 2-tuple `(keys, children)` vs. the leaf's 3-tuple `(keys, values, next_sibling)`.
- Source: entries/2026/05/29/topic-internal-node-format.md

### [ACCEPT] not-lehman-yao-blink-tree
The B-tree is a classical single-threaded B⁺-tree, not a Lehman-Yao B-link tree — internal nodes lack right-link pointers, there are no high keys, and no latch coupling; leaf sibling pointers exist solely for range scan traversal.
- Source: entries/2026/05/29/topic-internal-node-format.md

### [REJECT] leaf-sibling-pointer-present
Duplicates `leaf-wire-format-is-header-sibling-entries` and `btree-leaf-sibling-chain` which already describe the leaf sibling pointer field and sentinel.
- Source: entries/2026/05/29/topic-internal-node-format.md

### [REJECT] sibling-asymmetry-reflects-purpose
Duplicates `sibling-field-is-structural-only` which already establishes that sibling pointers serve range scans, not concurrency.
- Source: entries/2026/05/29/topic-internal-node-format.md

### [REJECT] free-list-is-intrusive-singly-linked
Duplicates `btree-free-list-intrusive` (exact same claim about intrusive singly-linked list with next pointer at HEADER_SIZE offset).
- Source: entries/2026/05/29/topic-intrusive-free-list-pattern.md

### [REJECT] allocate-page-prefers-reuse
Duplicates `allocate-page-prefers-free-list`.
- Source: entries/2026/05/29/topic-intrusive-free-list-pattern.md

### [ACCEPT] free-list-head-persisted-in-page-zero
The free list head pointer is stored as the fifth field of the metadata page (page 0) in the layout `[root_page:4B][height:4B][total_keys:4B][next_free_page:4B][free_list_head:4B]`, making it durable across restarts.
- Source: entries/2026/05/29/topic-intrusive-free-list-pattern.md

### [REJECT] free-page-not-atomic-with-meta-update
Duplicates `btree-free-page-bypasses-wal` which already captures that `free_page()` writes directly without WAL protection.
- Source: entries/2026/05/29/topic-intrusive-free-list-pattern.md

### [ACCEPT] pbft-sole-sort-keys-user
`byzantine-fault-tolerance/pbft.py:46` is the only call site in the codebase that uses `json.dumps(sort_keys=True)` for hashing; all other `json.dumps` calls are for storage serialization where canonical form is irrelevant.
- Source: entries/2026/05/29/topic-json-canonicalization-risks.md

### [ACCEPT] pbft-default-str-fragile
The `default=str` parameter in the PBFT digest function silently converts non-serializable objects to their `str()` representation, which is not deterministic across replicas for types like `datetime` or custom objects — a latent consensus-breaking bug if non-primitive request payloads are introduced.
- Source: entries/2026/05/29/topic-json-canonicalization-risks.md

### [ACCEPT] merkle-tree-hashes-raw-bytes
The Merkle tree implementation avoids serialization canonicalization entirely by accepting caller-provided `bytes` and hashing them directly via `hashlib.sha256(data)`, making it immune to the JSON encoding ambiguity that affects the PBFT digest.
- Source: entries/2026/05/29/topic-json-canonicalization-risks.md

### [REJECT] json-dumps-sort-keys-insufficient-for-floats
General Python/JSON fact about float-to-string non-determinism, not a codebase-specific invariant — belongs in documentation, not in the belief registry.
- Source: entries/2026/05/29/topic-json-canonicalization-risks.md


## Entry: topic-kafka-consumer-offset-semantics.md

### [ACCEPT] auto-commit-not-default
`Consumer` defaults to `auto_commit=False` (line 182), requiring explicit `commit()` calls; at-least-once semantics via auto-commit are opt-in, not the default behavior
- Source: entries/2026/05/29/topic-kafka-consumer-offset-semantics.md

### [ACCEPT] offset-tracking-per-partition
Consumer tracks offsets as a `dict[tuple[str, int], int]` mapping `(topic, partition)` to offset, so commit granularity is per-partition, not per-message
- Source: entries/2026/05/29/topic-kafka-consumer-offset-semantics.md

### [ACCEPT] no-transactional-offset-commit
The partitioned log provides no mechanism to atomically commit consumer offsets together with processing side effects, ruling out exactly-once delivery without external idempotency
- Source: entries/2026/05/29/topic-kafka-consumer-offset-semantics.md

### [ACCEPT] consumer-group-offset-resume
A new consumer joining a group loads committed offsets from the broker (tested at `test_partitioned_log.py:152`), enabling offset-based resume but also duplicate redelivery on crash
- Source: entries/2026/05/29/topic-kafka-consumer-offset-semantics.md

---

## Entry: topic-lamport-1978-paper.md

### [REJECT] lamport-receive-tick-uses-max-plus-one
Duplicate of existing `lamport-receive-tick-guarantees-causal-order`
- Source: entries/2026/05/29/topic-lamport-1978-paper.md

### [REJECT] lamport-clock-necessary-not-sufficient
Duplicate of existing `lamport-timestamp-order-not-implies-causality` and `lamport-happens-before-uses-graph-not-timestamps`
- Source: entries/2026/05/29/topic-lamport-1978-paper.md

### [REJECT] lamport-send-is-synchronous-simplification
Duplicate of existing `lamport-send-delivers-synchronously`
- Source: entries/2026/05/29/topic-lamport-1978-paper.md

---

## Entry: topic-lehman-yao-1981.md

### [ACCEPT] leaf-sibling-not-blink
Leaf `next_sibling` pointers serve range scans only, not Lehman & Yao concurrent-split recovery; internal nodes have no right-links, making this a standard B+ tree, not a B-link tree
- Source: entries/2026/05/29/topic-lehman-yao-1981.md

### [REJECT] no-concurrency-control
Duplicate of existing `btree-single-threaded-assumption` and `no-concurrency-primitives`
- Source: entries/2026/05/29/topic-lehman-yao-1981.md

### [ACCEPT] no-high-key-fence
No node stores a high-key (upper-bound fence key); searchers cannot detect mid-split state, which would be required for Lehman & Yao's concurrent search protocol
- Source: entries/2026/05/29/topic-lehman-yao-1981.md

### [REJECT] sibling-sentinel-convention
Trivial implementation detail already partially covered by `sibling-field-is-structural-only` which mentions the `NO_SIBLING = 0xFFFFFFFF` sentinel
- Source: entries/2026/05/29/topic-lehman-yao-1981.md

---

## Entry: topic-length-prefix-framing-resilience.md

### [ACCEPT] no-component-resyncs-after-corruption
Every binary record reader in this codebase (WAL, both Bitcasks, LSM WAL, SSTable, B-tree WAL) stops at the first CRC failure, short read, or parse error; none attempt to scan forward for the next valid record boundary
- Source: entries/2026/05/29/topic-length-prefix-framing-resilience.md

### [REJECT] lsm-wal-silent-cascading-misframes
Duplicate of existing `lsm-wal-has-no-integrity-check` and `lsm-and-sstable-have-no-checksums`
- Source: entries/2026/05/29/topic-length-prefix-framing-resilience.md

### [ACCEPT] event-store-ndjson-could-resync-but-doesnt
The event store uses newline-delimited JSON where per-line resync is architecturally possible, but `_load_from_file` halts on the first `json.JSONDecodeError` rather than skipping the bad line
- Source: entries/2026/05/29/topic-length-prefix-framing-resilience.md

### [ACCEPT] btree-page-alignment-isolates-corruption
The B-tree's fixed-size pages (`page_num * page_size` addressing) provide natural resync boundaries for the data file — corruption of one page does not affect reads of other pages, unlike the streaming WAL formats
- Source: entries/2026/05/29/topic-length-prefix-framing-resilience.md

### [ACCEPT] sstable-sparse-index-bounds-corruption
A corrupted SSTable data entry only affects lookups within one sparse index segment; the sparse index provides independent entry points into different file regions, bounding corruption blast radius to ~`block_size` entries
- Source: entries/2026/05/29/topic-length-prefix-framing-resilience.md

### [REJECT] contiguous-framing-prevents-resync
Duplicate of existing `wal-contiguous-no-block-alignment`
- Source: entries/2026/05/29/topic-length-prefix-framing-resilience.md

---

## Entry: topic-leveldb-log-format.md

### [REJECT] wal-py-stops-at-first-corruption
Duplicate of existing `wal-no-resync-after-corruption` and `wal-corruption-returns-valid-prefix`
- Source: entries/2026/05/29/topic-leveldb-log-format.md

### [REJECT] no-block-aligned-wal-in-codebase
Duplicate of existing `wal-contiguous-no-block-alignment`
- Source: entries/2026/05/29/topic-leveldb-log-format.md

### [REJECT] wal-py-crc-covers-optype-key-value
Duplicate of existing `all-three-implementations-use-payload-only-crc` and `wal-crc-does-not-cover-seqnum`
- Source: entries/2026/05/29/topic-leveldb-log-format.md


### [ACCEPT] sstable-merge-forward-only
The k-way merge in `sstable-and-compaction/sstable.py` is forward-only with no direction tracking or reverse iteration support; there is no direction enum, no `FindLargest`, and no heap rebuild on direction change.
- Source: entries/2026/05/29/topic-leveldb-merger.md

### [REJECT] heap-tuple-ordering-for-dedup
Overlaps with existing `sstable-merge-dedup-newest-timestamp` (newest timestamp wins) and `heap-tiebreak-uses-source-index` (source index as tiebreaker); the negation trick is an implementation detail, not an invariant.
- Source: entries/2026/05/29/topic-leveldb-merger.md

### [REJECT] no-leveldb-source-in-repo
Trivially obvious — this is a Python reference implementation repo, not a LevelDB fork; stating the absence of C++ source is not useful for working in the codebase.
- Source: entries/2026/05/29/topic-leveldb-merger.md

### [REJECT] compaction-avoids-direction-complexity
Design rationale comparing to LevelDB, not a testable codebase invariant; the actual constraint is captured by `sstable-merge-forward-only`.
- Source: entries/2026/05/29/topic-leveldb-merger.md

### [REJECT] wal-no-block-fragmentation
Duplicates existing `wal-contiguous-no-block-alignment` which already states the WAL uses variable-length records with no fixed block boundaries.
- Source: entries/2026/05/29/topic-leveldb-record-fragmentation.md

### [REJECT] wal-corruption-halts-recovery
Duplicates existing `wal-corruption-stops-per-file` (stops at first corrupt record within each file but continues to next file) and `wal-no-resync-after-corruption`.
- Source: entries/2026/05/29/topic-leveldb-record-fragmentation.md

### [REJECT] wal-large-records-no-limit
Direct consequence of existing `wal-contiguous-no-block-alignment`; not an independent invariant.
- Source: entries/2026/05/29/topic-leveldb-record-fragmentation.md

### [REJECT] leveldb-fragment-crash-safety
About LevelDB's design, not about this codebase; beliefs should be testable claims about the code a developer works in.
- Source: entries/2026/05/29/topic-leveldb-record-fragmentation.md

### [REJECT] setup-bottommost-scans-below-output-level
About RocksDB's `Compaction::SetupBottomMostLevel()` internals, not about this codebase; the relevant gap is already captured by `sstable-compaction-manager-lacks-bottommost-detection`.
- Source: entries/2026/05/29/topic-leveled-compaction-tombstone-safety.md

### [REJECT] bottommost-false-negative-is-safe
RocksDB design principle; the conservative correctness of this codebase is already captured by `compaction-manager-never-removes-tombstones`.
- Source: entries/2026/05/29/topic-leveled-compaction-tombstone-safety.md

### [REJECT] bottommost-enables-three-optimizations
About RocksDB's `ProcessKeyValueCompaction`, not about this codebase.
- Source: entries/2026/05/29/topic-leveled-compaction-tombstone-safety.md

### [REJECT] bottommost-level-enum-is-trigger-not-detection
About RocksDB's `BottommostLevelCompaction` enum, not about this codebase.
- Source: entries/2026/05/29/topic-leveled-compaction-tombstone-safety.md

### [REJECT] lcs-picks-most-overlap
Already exists as `lcs-picks-max-overlap-sstable`.
- Source: entries/2026/05/29/topic-leveled-compaction-write-amplification.md

### [ACCEPT] overlapping-determines-merge-set
The merge set for leveled compaction is always exactly one source SSTable plus all SSTables in the target level whose key range intersects it, constructed via `_overlapping()` at `sstable.py:428-429`.
- Source: entries/2026/05/29/topic-leveled-compaction-write-amplification.md

### [ACCEPT] compaction-is-explicit-not-background
Both the LSM and SSTable modules trigger compaction via explicit synchronous method calls (`compact()` / `run_compaction()`), not background threads — removing the write-amplification pressure that motivates least-overlap selection in production systems.
- Source: entries/2026/05/29/topic-leveled-compaction-write-amplification.md

### [ACCEPT] leveled-compaction-promotes-to-next-level
After leveled compaction, resulting SSTables are assigned `level = max_level + 1` where `max_level` is the highest level among inputs, establishing the level hierarchy (observable in test assertion at `test_sstable.py:119`).
- Source: entries/2026/05/29/topic-leveled-compaction-write-amplification.md

### [ACCEPT] linearizable-read-requires-consensus
`LinearizableRegister.read()` broadcasts through the TOB consensus layer rather than reading local state, because a node cannot distinguish "I have the latest value" from "I'm partitioned and stale" without majority confirmation.
- Source: entries/2026/05/29/topic-linearizable-reads-via-tob.md

### [ACCEPT] tob-uses-single-decree-paxos-per-slot
Each slot in the total order broadcast is decided by an independent `ConsensusInstance` running single-decree Paxos with prepare/accept phases (`total_order_broadcast.py:3`).
- Source: entries/2026/05/29/topic-linearizable-reads-via-tob.md

### [ACCEPT] no-leader-lease-in-codebase
Neither the Raft nor TOB implementations include leader lease or ReadIndex optimizations; all read and write operations pay full consensus cost.
- Source: entries/2026/05/29/topic-linearizable-reads-via-tob.md

### [ACCEPT] reads-and-writes-same-consensus-path
`LinearizableRegister` routes both reads and writes through the same TOB slot mechanism, giving them positions in the same total order — no read-only fast path exists.
- Source: entries/2026/05/29/topic-linearizable-reads-via-tob.md


### [ACCEPT] truncate-advances-base-offset-additively
`Topic.truncate` updates `_base_offsets[partition]` with `+= actual`, a relative shift, because it only removes a contiguous prefix from the front of the partition log.
- Source: entries/2026/05/29/topic-log-compaction-vs-retention.md

### [ACCEPT] compact-sets-base-offset-absolutely
`Topic.compact_partition` overwrites `_base_offsets[partition]` with the first surviving message's offset (absolute assignment), discarding whatever the previous base was, because compaction removes messages from arbitrary positions.
- Source: entries/2026/05/29/topic-log-compaction-vs-retention.md

### [ACCEPT] compaction-preserves-original-offsets
After `compact_partition`, surviving messages retain their original offset values, creating non-contiguous gaps in the offset sequence; offsets are never renumbered.
- Source: entries/2026/05/29/topic-log-compaction-vs-retention.md

### [ACCEPT] read-uses-linear-scan-for-offset-gaps
`Topic.read` finds the start position by scanning for `msg.offset >= offset` rather than computing an array index, which is necessary because compaction creates offset holes that break arithmetic indexing.
- Source: entries/2026/05/29/topic-log-compaction-vs-retention.md

### [ACCEPT] no-concurrency-protection-on-base-offsets
Neither `truncate` nor `compact_partition` holds a lock; concurrent calls on the same partition would race on both `_partitions` and `_base_offsets`, assuming single-threaded execution.
- Source: entries/2026/05/29/topic-log-compaction-vs-retention.md

### [ACCEPT] directory-scan-recovery-unsafe-during-compaction
`_load_existing_sstables` using `os.listdir` cannot distinguish pre-compaction from post-compaction file sets after a crash — if the crash occurs after creating the merged SSTable but before deleting the inputs, recovery sees duplicates; the inverse loses data.
- Source: entries/2026/05/29/topic-manifest-and-crash-recovery.md

### [REJECT] lsm-wal-covers-data-not-metadata
Duplicates existing belief `wal-protects-data-not-metadata` and overlaps with `lsm-no-manifest`.
- Source: entries/2026/05/29/topic-manifest-and-crash-recovery.md

### [REJECT] sstable-metadata-has-level-field-but-no-persistence
Duplicates existing belief `sstable-metadata-level-field-unused`, which already captures that the level field exists but has no operational use.
- Source: entries/2026/05/29/topic-manifest-and-crash-recovery.md

### [REJECT] wal-module-has-batch-atomicity-pattern
Duplicates existing belief `wal-batch-atomicity-via-commit-record`, which already describes the COMMIT-marker pattern for atomic batch writes.
- Source: entries/2026/05/29/topic-manifest-and-crash-recovery.md

### [REJECT] btree-no-delete-operation
Contradicted by multiple existing beliefs (`btree-delete-skips-free-page`, `btree-empty-leaf-freed-on-delete`, `btree-delete-no-commit-on-miss`, `btree-leaf-sibling-patched-on-remove`) that describe delete behavior in detail. The entry's grep likely missed the method.
- Source: entries/2026/05/29/topic-merge-vs-lazy-free.md

### [REJECT] btree-no-merge-or-redistribute
Duplicates existing beliefs `btree-impl-asymmetric-split-vs-merge` and `btree-sibling-pointers-unused-for-merge`, which already establish that merge/redistribute logic is absent.
- Source: entries/2026/05/29/topic-merge-vs-lazy-free.md

### [REJECT] btree-free-list-unused-by-logic
Contradicted by existing belief `btree-empty-leaf-freed-on-delete`, which states that empty leaves are returned to the free list via `free_page()`.
- Source: entries/2026/05/29/topic-merge-vs-lazy-free.md

### [REJECT] btree-next-sibling-for-range-scans
Duplicates existing beliefs `range-scan-follows-sibling-chain` (range scans use sibling pointers) and `btree-sibling-pointers-unused-for-merge` (sibling pointers not used for merge).
- Source: entries/2026/05/29/topic-merge-vs-lazy-free.md

### [ACCEPT] merkle-proof-size-is-log-n
A Merkle proof contains exactly `height` sibling hashes where height = log2(next_power_of_2(leaf_count)), making both proof size and verification cost O(log N).
- Source: entries/2026/05/29/topic-merkle-proof-security-model.md

### [REJECT] verify-proof-is-stateless
Duplicates existing belief `verify-proof-is-pure-static`, which already captures that `verify_proof` is a static method requiring no tree instance.
- Source: entries/2026/05/29/topic-merkle-proof-security-model.md

### [REJECT] root-hash-trust-is-external
Duplicates existing belief `verify-proof-root-trust`, which already states that `verify_proof` does not authenticate the root and callers must obtain a trusted root independently.
- Source: entries/2026/05/29/topic-merkle-proof-security-model.md

### [ACCEPT] sibling-direction-matters
Proof siblings are tagged "left" or "right" because SHA-256 concatenation is order-dependent — `H(A||B) != H(B||A)` — so swapping sibling position produces a different parent hash and verification fails.
- Source: entries/2026/05/29/topic-merkle-proof-security-model.md

### [ACCEPT] padding-leaves-use-empty-hash
Non-power-of-2 leaf counts are padded with `EMPTY_HASH` (SHA-256 of empty bytes) to form a complete binary tree; these padding nodes appear in proof paths but do not represent real data.
- Source: entries/2026/05/29/topic-merkle-proof-security-model.md

### [ACCEPT] wal-seq-num-global-monotonic
`_seq_num` is a single in-memory counter that only increments via `+= 1`; all records across all WAL files share one monotonically increasing sequence space that never resets, even across file rotation.
- Source: entries/2026/05/29/topic-multi-file-replay-ordering.md

### [REJECT] wal-rotation-after-write
Duplicates existing belief `wal-rotation-is-post-write`.
- Source: entries/2026/05/29/topic-multi-file-replay-ordering.md

### [REJECT] wal-batch-never-spans-files
Duplicates existing belief `wal-batch-single-file`.
- Source: entries/2026/05/29/topic-multi-file-replay-ordering.md

### [REJECT] wal-filename-sort-equals-creation-order
Duplicates existing belief `wal-segment-naming-zero-padded`, which already establishes that zero-padded filenames ensure lexicographic = numeric ordering.
- Source: entries/2026/05/29/topic-multi-file-replay-ordering.md

### [ACCEPT] wal-rotation-monotonicity-untested
`test_rotation` asserts record count after rotation but does not verify that sequence numbers are monotonically increasing across file boundaries — the cross-file monotonicity invariant holds by construction but has no regression test.
- Source: entries/2026/05/29/topic-multi-file-replay-ordering.md


### [ACCEPT] tob-full-paxos-per-slot
Every slot in Total Order Broadcast runs the complete two-phase Paxos protocol (Prepare→Promise→Accept→Accepted) from scratch; no state carries between slots to skip Phase 1
- Source: entries/2026/05/29/topic-multi-paxos-vs-single-decree.md

### [ACCEPT] tob-no-leader-concept
TOBNode has no leader election or lease mechanism; any node can propose for any undecided slot at any time, making it vulnerable to competing proposals
- Source: entries/2026/05/29/topic-multi-paxos-vs-single-decree.md

### [ACCEPT] tob-proposal-number-encodes-node-id
Proposal numbers are computed as `round * num_nodes + node_id`, guaranteeing uniqueness across nodes within the same round but restarting at round 0 for each slot independently
- Source: entries/2026/05/29/topic-multi-paxos-vs-single-decree.md

### [ACCEPT] raft-is-multi-paxos-optimization
The Raft implementation in `raft-consensus/raft.py` embodies the Multi-Paxos leader-lease pattern: one election per term via `_become_leader`, then Phase-2-only replication (AppendEntries) for all entries within that term — heartbeats act as the lease
- Source: entries/2026/05/29/topic-multi-paxos-vs-single-decree.md

### [ACCEPT] tob-competing-proposals-bump-rounds
When a TOB slot is decided by another proposer with a different value, the losing node re-proposes its value for a new slot rather than retrying the same slot with a higher round
- Source: entries/2026/05/29/topic-multi-paxos-vs-single-decree.md

### [ACCEPT] lsm-no-concurrency-control
The LSM tree in `lsm.py` has no locking, reference counting, or Version snapshots; `compact()` mutates `self._sstables` in place, so concurrent readers could see inconsistent state or reference deleted SSTables
- Source: entries/2026/05/29/topic-mvcc-version-snapshots.md

### [REJECT] lsm-compaction-replaces-all
Duplicates existing belief `lsm-compaction-is-full-merge` which already captures that compact() merges all SSTables into one
- Source: entries/2026/05/29/topic-mvcc-version-snapshots.md

### [ACCEPT] mvcc-pattern-exists-at-row-level
The codebase implements MVCC visibility (`ssi_database.py:_visible_value`, `mvcc_database.py:Version`) at the row/key level but not at the SSTable-file level where LevelDB uses Version reference counting for concurrent reader isolation
- Source: entries/2026/05/29/topic-mvcc-version-snapshots.md

### [ACCEPT] sstable-metadata-has-level-field
`SSTableMetadata` in `sstable-and-compaction/sstable.py` tracks a `level` field, indicating the compaction module is structured for leveled compaction even though `lsm.py` uses flat single-level merges
- Source: entries/2026/05/29/topic-mvcc-version-snapshots.md

### [REJECT] python-bytes-never-page-aligned
General CPython runtime implementation detail, not a testable claim about this codebase
- Source: entries/2026/05/29/topic-o-direct-aligned-io.md

### [REJECT] o-direct-requires-three-alignments
General OS/hardware constraint, not specific to this codebase; the relevant codebase fact (`no-o-direct-usage`) is already captured
- Source: entries/2026/05/29/topic-o-direct-aligned-io.md

### [REJECT] python-open-allocates-unaligned-buffers
General CPython runtime behavior, not a testable claim about this codebase
- Source: entries/2026/05/29/topic-o-direct-aligned-io.md

### [REJECT] mmap-returns-page-aligned-by-definition
General OS-level invariant, not specific to this codebase
- Source: entries/2026/05/29/topic-o-direct-aligned-io.md

### [ACCEPT] page-manager-4096-coincidence
`PageManager` defaults to 4096-byte pages matching the OS page size and common disk sector size, but this is a coincidental match with `O_DIRECT` alignment requirements — the implementation uses buffered I/O through Python's `open()` and never enforces alignment
- Source: entries/2026/05/29/topic-o-direct-aligned-io.md

### [ACCEPT] orset-tombstone-set-monotonic
`ORSet._tombstones` is append-only: tags are added during `remove()` and unioned during `merge()` but never deleted, so the tombstone set grows without bound over the lifetime of the replica
- Source: entries/2026/05/29/topic-or-set-tombstone-growth.md

### [ACCEPT] orset-no-causal-context
The ORSet implementation uses no version vector, dot context, or causal summary to compress tombstone state; each removed tag is stored individually as a `(replica_id, seq)` tuple in the tombstone set
- Source: entries/2026/05/29/topic-or-set-tombstone-growth.md

### [ACCEPT] orset-tombstone-required-for-merge-correctness
Dropping ORSet tombstones would cause removed elements to reappear when merging with a replica that still holds the element's active tags — the tombstone set in `merge()` at line 165 suppresses stale tags via set difference
- Source: entries/2026/05/29/topic-or-set-tombstone-growth.md

### [REJECT] lsm-tombstone-gc-not-applicable-to-crdts
Covered by combination of existing beliefs `distributed-tombstone-removal-needs-replication-convergence` and `lsm-compact-removes-tombstones-safely`
- Source: entries/2026/05/29/topic-or-set-tombstone-growth.md

### [REJECT] lsm-wal-no-fsync
Duplicates existing belief `lsm-wal-has-no-fsync`
- Source: entries/2026/05/29/topic-os-fsync-durability.md

### [REJECT] lsm-truncate-before-sstable-sync
Duplicates existing belief `lsm-flush-double-loss-window` which captures the same crash scenario where both WAL and SSTable data can be lost
- Source: entries/2026/05/29/topic-os-fsync-durability.md

### [REJECT] wal-module-syncs-every-write
Duplicates existing belief `wal-fsync-per-entry`
- Source: entries/2026/05/29/topic-os-fsync-durability.md

### [REJECT] btree-wal-fsync-pagefile-deferred
Covered by existing beliefs `btree-wal-fsync-per-entry` (WAL fsyncs per entry) and `write-meta-no-fsync` (PageManager defers sync)
- Source: entries/2026/05/29/topic-os-fsync-durability.md

### [ACCEPT] event-store-persist-no-durability
`EventStore._persist_event()` in `event-sourcing-store/event_store.py` opens the file in a `with` block, writes JSON, and relies on implicit `close()` — no explicit `flush()` or `os.fsync()`, making persisted events vulnerable to loss on OS crash
- Source: entries/2026/05/29/topic-os-fsync-durability.md


### [ACCEPT] leaf-serialized-overhead-is-7-bytes
Every leaf page has a fixed 7-byte overhead (3B header via `HEADER_FMT = '>BH'` + 4B `next_sibling` pointer) before any key-value entries, setting the usable capacity to `page_size - 7` bytes.
- Source: entries/2026/05/29/topic-page-overflow-and-split-mechanics.md

### [ACCEPT] leaf-entry-cost-is-variable
Each leaf entry costs `4 + len(key_bytes) + len(value_bytes)` bytes (2B key length + key + 2B value length + value), so maximum keys per page depends on actual data sizes, not just a fixed `max_keys` count.
- Source: entries/2026/05/29/topic-page-overflow-and-split-mechanics.md

### [ACCEPT] page-size-default-is-4096
`PageManager` defaults to 4096-byte pages, matching typical filesystem block size; this sets the hard upper bound for serialized leaf and internal node content.
- Source: entries/2026/05/29/topic-page-overflow-and-split-mechanics.md

### [ACCEPT] gossip-fixed-three-threshold-fsm
Failure detection uses exactly three fixed time thresholds (`t_suspect=5`, `t_dead=10`, `t_cleanup=20`) producing a deterministic state machine with no runtime adaptation to network conditions.
- Source: entries/2026/05/29/topic-phi-accrual-failure-detector.md

### [ACCEPT] gossip-no-arrival-history
Each node stores only the most recent `timestamp_last_updated` per peer, discarding all inter-arrival time history that would be needed for probabilistic failure detection (e.g., phi accrual).
- Source: entries/2026/05/29/topic-phi-accrual-failure-detector.md

### [ACCEPT] gossip-uniform-thresholds
All nodes in a `GossipCluster` receive identical threshold values from the cluster constructor at `__init__` (line 117); individual nodes cannot calibrate sensitivity to their specific network path.
- Source: entries/2026/05/29/topic-phi-accrual-failure-detector.md

### [ACCEPT] gossip-detect-failures-is-stateless
`detect_failures` makes decisions using only `current_time - timestamp_last_updated`; it consults no historical distribution or sliding window, making it a pure point-in-time comparison rather than a statistical inference.
- Source: entries/2026/05/29/topic-phi-accrual-failure-detector.md

### [REJECT] fsync-semantics-vary-by-filesystem
General OS/filesystem knowledge (ext4 vs XFS vs btrfs behavior), not a testable claim about this codebase's code.
- Source: entries/2026/05/29/topic-posix-durability-guarantees.md

### [ACCEPT] fdatasync-safe-for-all-append-paths
Every append-only file in the codebase (WAL segments, Bitcask data files) writes sequentially with monotonically increasing file size, making `fdatasync` a safe drop-in for `fsync` on those paths; B-tree in-place page overwrites benefit even more since file size doesn't change.
- Source: entries/2026/05/29/topic-posix-durability-guarantees.md

### [REJECT] posix-fsync-is-per-file-not-cross-file
General POSIX specification knowledge, not a testable claim about this codebase; the cross-file ordering consequence is already captured by `btree-wal-before-data` and related beliefs.
- Source: entries/2026/05/29/topic-posix-durability-guarantees.md

### [ACCEPT] macos-fsync-not-durable
On the development platform (Darwin/APFS), `os.fsync()` may not flush the disk write cache; true durability requires `fcntl(fd, F_FULLFSYNC)`, which is never used at any of the 13 fsync call sites in the codebase.
- Source: entries/2026/05/29/topic-posix-durability-guarantees.md

### [REJECT] btree-no-concurrent-access-control
Duplicate of existing beliefs `btree-single-threaded-assumption`, `btree-single-writer`, and `no-concurrency-primitives`.
- Source: entries/2026/05/29/topic-postgres-nbtree-readme.md

### [REJECT] btree-leaf-sibling-not-updated-on-delete
Contradicts existing accepted belief `btree-leaf-sibling-patched-on-remove`, which confirms that the predecessor leaf's `next_sibling` IS updated when a leaf is freed; the bug described in the entry was fixed.
- Source: entries/2026/05/29/topic-postgres-nbtree-readme.md

### [REJECT] btree-split-uses-recursive-return
Duplicate of existing belief `btree-recursive-insert-returns-union`, which captures that `_insert` returns either `'inserted'` or `(mid_key, new_page_num)` to propagate splits upward.
- Source: entries/2026/05/29/topic-postgres-nbtree-readme.md

### [ACCEPT] btree-wal-uses-full-page-images
The WAL logs complete page images (physical logging) rather than logical operations, which eliminates incomplete-split states during recovery but increases write amplification compared to PostgreSQL's logical WAL entries.
- Source: entries/2026/05/29/topic-postgres-nbtree-readme.md

### [REJECT] btree-free-list-immediate-reclaim
Covered by the combination of `btree-empty-leaf-freed-on-delete` (pages are freed immediately) and `btree-single-threaded-assumption` (no concurrent access makes deferred reclamation unnecessary).
- Source: entries/2026/05/29/topic-postgres-nbtree-readme.md

### [REJECT] btree-delete-never-frees-pages
Contradicts existing accepted belief `btree-empty-leaf-freed-on-delete`; the Bug 5 described in `fix-plan.md` has been fixed and empty leaves are now freed via `PageManager.free_page()`.
- Source: entries/2026/05/29/topic-postgres-nbtree-vacuum.md

### [REJECT] btree-no-concurrency-control
Duplicate of existing beliefs `btree-single-threaded-assumption` and `no-concurrency-primitives`.
- Source: entries/2026/05/29/topic-postgres-nbtree-vacuum.md

### [REJECT] btree-free-list-exists-unused
Contradicts existing accepted belief `free-page-only-called-from-delete`, which confirms the free list IS called from the delete path; the code was updated after the entry was written.
- Source: entries/2026/05/29/topic-postgres-nbtree-vacuum.md

### [ACCEPT] btree-leaf-sibling-chain-forward-only
Leaf pages store a `next_sibling` forward pointer (`NO_SIBLING = 0xFFFFFFFF` sentinel) but no backward pointer, so unlinking a leaf from the sibling chain requires locating the predecessor via parent traversal rather than direct backlink.
- Source: entries/2026/05/29/topic-postgres-nbtree-vacuum.md


## Belief Proposals

### Entry: topic-projection-snapshot-interaction.md

### [ACCEPT] live-projection-no-snapshot-interval
`LiveProjection` is never constructed with a `snapshot_interval`, so the automatic snapshot path in `catch_up()` never triggers for live projections
- Source: entries/2026/05/29/topic-projection-snapshot-interaction.md

### [ACCEPT] snapshot-state-excludes-subscription
Snapshot data (`_store._snapshots`) contains only `state` and `position`; subscriber callbacks registered in `EventStore._subscribers` are not captured, so restoring a snapshot does not restore live-update behavior
- Source: entries/2026/05/29/topic-projection-snapshot-interaction.md

### [ACCEPT] live-projection-full-replay-on-restart
A `LiveProjection` must replay all events from position 0 on initialization because there is no snapshot restore path wired into its construction
- Source: entries/2026/05/29/topic-projection-snapshot-interaction.md

---

### Entry: topic-pytest-default-collection.md

### [REJECT] tester-test-files-not-discovered
Duplicate of existing belief `tester-naming-avoids-pytest-discovery`
- Source: entries/2026/05/29/topic-pytest-default-collection.md

### [ACCEPT] write-skew-has-no-default-tests
The `write-skew-detection` module's tests exist only in `tester_test_ssi.py`, making it entirely untested under default `pytest` invocation since that filename does not match the `test_*.py` / `*_test.py` globs
- Source: entries/2026/05/29/topic-pytest-default-collection.md

### [ACCEPT] tester-versions-have-more-test-functions
The `tester_test_*.py` files consistently contain more test functions than their `test_*.py` counterparts (bloom-filter: 11 vs 5, gossip-protocol: 12 vs 10), and include tests for features not covered in the default-discovered files
- Source: entries/2026/05/29/topic-pytest-default-collection.md
- **Note:** Contradicts existing belief `pytest-files-have-broader-coverage` which claims `test_*.py` files have broader coverage based on line count; the two beliefs measure different things (function count vs line count) but reach opposite conclusions about which suite is more comprehensive

---

### Entry: topic-raft-leader-stickiness.md

### [ACCEPT] raft-no-prevote-or-lease
The Raft implementation has no PreVote, leader lease, or CheckQuorum mechanism; split-brain safety relies entirely on term numbers and the single-vote-per-term invariant
- Source: entries/2026/05/29/topic-raft-leader-stickiness.md

### [ACCEPT] raft-stale-leader-accepts-writes
A partitioned leader continues accepting client requests (returning `success: True`) even though those entries can never commit and will be overwritten when the partition heals — the gap a leader lease would close
- Source: entries/2026/05/29/topic-raft-leader-stickiness.md

### [ACCEPT] raft-rejoin-forces-election
A partitioned node that rejoins after repeated failed elections carries an inflated term number, forcing the healthy leader to step down via `_become_follower` and triggering a completely unnecessary cluster-wide election
- Source: entries/2026/05/29/topic-raft-leader-stickiness.md

### [ACCEPT] raft-randomized-timeout-only-split-vote-defense
The only mechanism preventing simultaneous candidacy is the randomized election timeout range (default 150–300ms); there is no protocol-level tiebreaker such as PreVote
- Source: entries/2026/05/29/topic-raft-leader-stickiness.md

---

### Entry: topic-raft-log-compaction.md

### [ACCEPT] raft-log-never-shrinks
`self._log` in `RaftNode` is append-only: no method ever removes entries, so the log grows monotonically with client requests and memory usage is unbounded
- Source: entries/2026/05/29/topic-raft-log-compaction.md

### [ACCEPT] raft-log-direct-indexed
Log entries are accessed by log index as a direct Python list index (`self._log[prev_log_index]`), which assumes all entries from index 0 onward are present — log truncation would require an offset or a different data structure
- Source: entries/2026/05/29/topic-raft-log-compaction.md

### [ACCEPT] raft-no-state-machine-apply
`_last_applied` is tracked but never used to actually apply commands to a state machine, making snapshotting impossible without first implementing application logic
- Source: entries/2026/05/29/topic-raft-log-compaction.md

### [ACCEPT] wal-has-truncation-raft-does-not
The WAL module provides explicit `truncate(up_to_seq)` and file rotation for log lifecycle management; the Raft module has no equivalent despite both being append-only log abstractions
- Source: entries/2026/05/29/topic-raft-log-compaction.md

### [ACCEPT] raft-sentinel-entry-assumption
The Raft log is initialized with a sentinel `LogEntry(term=0, index=0, command=None)` at position 0, and multiple methods depend on this entry always being present at `self._log[0]`
- Source: entries/2026/05/29/topic-raft-log-compaction.md

---

### Entry: topic-range-scan-implementation.md

### [REJECT] leaf-page-format-sibling-after-header
Duplicate of existing belief `leaf-wire-format-is-header-sibling-entries`
- Source: entries/2026/05/29/topic-range-scan-implementation.md

### [ACCEPT] range-scan-end-key-exclusive
`range_scan(start_key, end_key)` treats `end_key` as an exclusive upper bound — `range_scan("k05", "k10")` returns keys `k05` through `k09`
- Source: entries/2026/05/29/topic-range-scan-implementation.md

### [ACCEPT] range-scan-supports-unbounded
Calling `range_scan(start_key)` without an `end_key` scans from `start_key` through the last key in the rightmost leaf, terminating at the `NO_SIBLING` sentinel
- Source: entries/2026/05/29/topic-range-scan-implementation.md

---

**Summary:** 15 accepted, 5 rejected (4 duplicates, 1 already covered). One contradiction flagged between `tester-versions-have-more-test-functions` and existing `pytest-files-have-broader-coverage`.


### [ACCEPT] fencing-token-counter-is-global
`LockService._counter` is shared across all lock names, so fencing tokens are globally monotonic — a single `FencedResourceServer` can validate tokens from different locks without per-lock tracking.
- Source: entries/2026/05/29/topic-redlock-controversy.md

### [REJECT] fenced-server-rejects-strictly-lower-tokens
Duplicate of `fencing-rejects-stale-writes` which already captures the `<` (not `<=`) comparison and equal-token acceptance.
- Source: entries/2026/05/29/topic-redlock-controversy.md

### [ACCEPT] resource-token-tracking-is-independent
Each resource in `FencedResourceServer` maintains its own `_highest_token` entry, so a high token on resource A does not cause rejection of a lower token on resource B.
- Source: entries/2026/05/29/topic-redlock-controversy.md

### [REJECT] renewal-preserves-token-identity
Duplicate of `renew-preserves-token-number`.
- Source: entries/2026/05/29/topic-redlock-controversy.md

### [REJECT] unfenced-server-exists-as-negative-example
Duplicate of `unfenced-server-accepts-all`.
- Source: entries/2026/05/29/topic-redlock-controversy.md

### [REJECT] wal-records-contain-only-after-images
Duplicate of `wal-has-no-before-images`.
- Source: entries/2026/05/29/topic-redo-vs-undo-logging.md

### [REJECT] wal-batch-atomicity-via-commit-marker
Duplicate of `wal-batch-atomicity-via-commit-record`.
- Source: entries/2026/05/29/topic-redo-vs-undo-logging.md

### [REJECT] wal-checkpoint-bounds-replay
Covered by `checkpoint-record-not-used-in-recovery` (caller must supply seq externally) and `replay-strict-greater-than-filter` (replay skips records at or below the supplied seq).
- Source: entries/2026/05/29/topic-redo-vs-undo-logging.md

### [ACCEPT] wal-seq-num-is-monotonic-not-page-lsn
WAL `seq_num` increases monotonically across all records but is not stamped onto data pages, so there is no mechanism to detect whether a replayed operation was already applied — making replay non-idempotent against the underlying store.
- Source: entries/2026/05/29/topic-redo-vs-undo-logging.md

### [ACCEPT] hash-index-bitcask-shared-read-handles
`hash-index-storage/bitcask.py` uses a single cached file handle per segment for all reads via `_get_reader()`, making concurrent reads to the same segment unsafe due to shared seek position.
- Source: entries/2026/05/29/topic-reference-counted-file-handles.md

### [REJECT] log-structured-bitcask-fresh-handle-per-get
Duplicate of `bitcask-get-opens-fresh-handle`.
- Source: entries/2026/05/29/topic-reference-counted-file-handles.md

### [ACCEPT] python-bitcask-no-refcount-or-locking
Neither Python Bitcask implementation has reference counting, reader registration, or file-handle locking — compaction can delete segment files while a concurrent reader holds a stale index entry, unlike Erlang Bitcask which defers deletion until the last reader's refcount drops to zero.
- Source: entries/2026/05/29/topic-reference-counted-file-handles.md

### [REJECT] python-bitcask-single-threaded-scope
Intent/opinion claim ("deliberate scope boundary, not a bug"); the technical fact is already captured by `compact-single-threaded-assumption`.
- Source: entries/2026/05/29/topic-reference-counted-file-handles.md

### [ACCEPT] all-to-all-converges-in-one-round
`ALL_TO_ALL` topology delivers every pending change to every other node in a single `sync()` call, achieving convergence in one round when no custom-merge cascades occur.
- Source: entries/2026/05/29/topic-ring-vs-all-to-all-propagation-delay.md

### [ACCEPT] ring-requires-n-minus-one-rounds
`RING` topology advances changes by exactly one hop per `sync()` round due to snapshot-isolated pending queue draining, requiring N-1 rounds for a single-source change to reach all N nodes.
- Source: entries/2026/05/29/topic-ring-vs-all-to-all-propagation-delay.md

### [ACCEPT] ring-requeues-via-apply-remote-change
`apply_remote_change` appends accepted changes to the receiving node's `_pending` list, enabling store-and-forward propagation in ring topology; without this re-enqueue, changes would stop at the first hop.
- Source: entries/2026/05/29/topic-ring-vs-all-to-all-propagation-delay.md

### [ACCEPT] topology-does-not-change-conflict-outcome
Both topologies use the same deterministic `(timestamp, node_id)` comparison for LWW resolution; topology affects when and where conflicts are detected, not which value wins.
- Source: entries/2026/05/29/topic-ring-vs-all-to-all-propagation-delay.md

### [REJECT] longer-convergence-widens-conflict-window
Analytical consequence derivable from convergence round counts, not a directly testable code claim.
- Source: entries/2026/05/29/topic-ring-vs-all-to-all-propagation-delay.md

### [REJECT] lsm-compact-always-removes-tombstones
Covered by `lsm-compact-removes-tombstones-safely` and `lsm-flat-compaction-always-bottommost`.
- Source: entries/2026/05/29/topic-rocksdb-bottommost-level-enum.md

### [REJECT] no-rocksdb-bottommost-level-enum
Trivial negative claim; `sstable-compaction-manager-lacks-bottommost-detection` already captures the important absence.
- Source: entries/2026/05/29/topic-rocksdb-bottommost-level-enum.md

### [REJECT] merge-sstables-tombstone-flag
Duplicate of `merge-sstables-tombstone-flag-is-caller-controlled`.
- Source: entries/2026/05/29/topic-rocksdb-bottommost-level-enum.md

### [REJECT] lsm-compaction-is-flat-merge
Duplicate of `lsm-compaction-is-full-merge` and `lsm-tree-has-no-level-concept`.
- Source: entries/2026/05/29/topic-rocksdb-bottommost-level-enum.md


### [ACCEPT] lsm-compaction-last-writer-wins
Both LSM implementations resolve key conflicts during compaction by keeping only the newest value; older values are unconditionally discarded with no user-defined merge logic
- Source: entries/2026/05/29/topic-rocksdb-merge-operator.md

### [ACCEPT] sstable-entry-types-binary
The SSTable format supports exactly two entry types (value and tombstone via `TOMBSTONE_MARKER`); there is no third type for merge operands, which would be required for RocksDB-style merge support
- Source: entries/2026/05/29/topic-rocksdb-merge-operator.md

### [REJECT] compaction-requires-full-read-for-update
General CS fact about read-modify-write workloads, not a testable claim about this codebase's behavior
- Source: entries/2026/05/29/topic-rocksdb-merge-operator.md

### [REJECT] partial-merge-avoids-cross-level-reads
Describes RocksDB's MergeOperator design, not a claim about this codebase which has no merge operator implementation
- Source: entries/2026/05/29/topic-rocksdb-merge-operator.md

### [ACCEPT] macos-fsync-weaker-than-implied
The codebase uses `os.fsync()` exclusively, which on macOS/APFS does not flush the disk write cache; true durability requires `fcntl(fd, F_FULLFSYNC)` which is never used anywhere
- Source: entries/2026/05/29/topic-shadow-paging-or-steal-no-force.md

### [ACCEPT] sibling-pointers-leaf-only
Sibling pointers exist only on leaf nodes (`_serialize_leaf` at line 182); internal nodes have no right-links, making this a B+-tree, not a B-link tree
- Source: entries/2026/05/29/topic-sibling-pointer-maintenance.md

### [ACCEPT] no-high-key-per-node
Nodes store no upper-bound "high key," so a reader cannot detect that a concurrent split moved keys to a right sibling; this is the second missing prerequisite for Lehman-Yao's single-latch protocol
- Source: entries/2026/05/29/topic-sibling-pointer-maintenance.md

### [REJECT] wal-does-not-log-metadata
Duplicate of existing belief `btree-metadata-bypasses-wal`
- Source: entries/2026/05/29/topic-sibling-pointer-maintenance.md

### [ACCEPT] split-chain-rewiring-order
During a leaf split, the new right page is written with the old leaf's `next_sibling` before the old leaf is rewritten to point to the new page; WAL ordering ensures the pointer target is durable before anything references it
- Source: entries/2026/05/29/topic-sibling-pointer-maintenance.md

### [ACCEPT] single-threaded-by-design
`BTree` has zero synchronization primitives; the sibling chain, split logic, and page allocation are correct only because no concurrent reader or writer can observe intermediate states
- Source: entries/2026/05/29/topic-sibling-pointer-maintenance.md

### [REJECT] lsm-size-tiered-missing-key-probes-all
Duplicate of existing belief `lsm-get-probes-all-sstables-on-miss`
- Source: entries/2026/05/29/topic-size-tiered-vs-leveled-read-amplification.md

### [REJECT] leveled-compaction-one-sstable-per-level
Contradicts existing belief `lcs-produces-single-output-sstable` which documents that the implementation merges all inputs into one SSTable, violating the non-overlapping invariant at levels above L0
- Source: entries/2026/05/29/topic-size-tiered-vs-leveled-read-amplification.md

### [REJECT] bloom-filter-not-integrated-into-lsm
Duplicate of existing belief `bloom-filter-not-integrated`
- Source: entries/2026/05/29/topic-size-tiered-vs-leveled-read-amplification.md

### [REJECT] sstable-sparse-index-bounds-per-table-cost
Composite of existing beliefs `sstable-lookup-is-two-phase` (per-table cost) and `lsm-get-probes-all-sstables-on-miss` (cross-table count)
- Source: entries/2026/05/29/topic-size-tiered-vs-leveled-read-amplification.md

### [ACCEPT] compaction-threshold-controls-overlap-window
The `compaction_threshold` parameter (`lsm.py:204`) directly controls how many overlapping SSTables can accumulate before compaction, setting the worst-case missing-key probe count to `threshold - 1`
- Source: entries/2026/05/29/topic-size-tiered-vs-leveled-read-amplification.md

### [ACCEPT] join-uses-symmetric-interval-not-tumbling
`TimeWindow.contains()` uses `abs(t1 - t2) <= duration`, making join matching a symmetric interval check centered on each event, not aligned to any fixed time grid
- Source: entries/2026/05/29/topic-sliding-vs-tumbling-windows.md

### [REJECT] tumbling-aggregator-assigns-each-result-to-one-window
Duplicate of existing belief `tumbling-window-floor-division`
- Source: entries/2026/05/29/topic-sliding-vs-tumbling-windows.md

### [ACCEPT] join-window-and-aggregation-window-are-independent
The join `TimeWindow` duration controls which event pairs can match, while the `TumblingWindowAggregator` window size controls result grouping; these are independent parameters that can differ without violating any invariant
- Source: entries/2026/05/29/topic-sliding-vs-tumbling-windows.md

### [ACCEPT] expiration-cutoff-is-one-sided-despite-symmetric-matching
`_expire_events` uses `watermark - duration` as a one-sided cutoff for buffer cleanup, while `contains()` is symmetric; this is correct because future events can only arrive at or after the watermark
- Source: entries/2026/05/29/topic-sliding-vs-tumbling-windows.md


### [REJECT] sloppy-quorum-counts-hints-as-acks
Duplicate of existing `hinted-handoff-sloppy-quorum-counts-hints`
- Source: entries/2026/05/29/topic-sloppy-quorum-tradeoffs.md

### [REJECT] reads-never-consult-hint-nodes
Duplicate of existing `hinted-handoff-reads-skip-non-preferred`
- Source: entries/2026/05/29/topic-sloppy-quorum-tradeoffs.md

### [ACCEPT] dynamo-cluster-hints-after-quorum
`DynamoCluster.put` only stores hints after the write quorum is already met on real nodes, unlike `HintedHandoffStore` which counts hints toward the quorum — making DynamoCluster a strict-quorum-with-opportunistic-hints design, not a true sloppy quorum
- Source: entries/2026/05/29/topic-sloppy-quorum-tradeoffs.md

### [ACCEPT] hint-expiry-bounds-durability
Hints in `HintedHandoffStore` have a TTL; if `trigger_handoff` doesn't run before `created_at + hint_ttl`, `expire_hints` silently drops the hint, permanently losing that replica's copy of the data
- Source: entries/2026/05/29/topic-sloppy-quorum-tradeoffs.md

### [ACCEPT] mvcc-no-persistence
`MVCCDatabase` has zero serialization or persistence support; all state (`_versions`, `_transactions`, `_committed`, `_aborted`, counters) lives only in memory with no `to_dict`, `from_dict`, or file I/O of any kind
- Source: entries/2026/05/29/topic-snapshot-disk-persistence.md

### [ACCEPT] mvcc-visibility-depends-on-tx-metadata
`_is_visible` requires `_committed`, `_aborted`, and each transaction's `active_at_start` set; persisting version chains without this metadata makes them unreadable after restore since visibility cannot be determined
- Source: entries/2026/05/29/topic-snapshot-disk-persistence.md

### [ACCEPT] mvcc-counters-must-be-monotonic
`_next_tx_id` and `_next_timestamp` must never reissue a previously used value; if they reset to 1 on restart, visibility comparisons like `created_by < tx.tx_id` silently produce wrong results by conflating old and new transaction IDs
- Source: entries/2026/05/29/topic-snapshot-disk-persistence.md

### [ACCEPT] event-store-persist-pattern
`EventStore` in `event-sourcing-store/event_store.py` is the repo's existing model for optional disk persistence: accept `persist_path` in constructor, call `_load_from_file` on init if the file exists, call `_persist_event` on every mutation
- Source: entries/2026/05/29/topic-snapshot-disk-persistence.md

### [ACCEPT] mvcc-active-tx-not-recoverable
Active (uncommitted) transactions cannot survive a process restart; the safe recovery strategy is to treat them as aborted, which is consistent with `_is_visible`'s existing handling of aborted transaction versions as invisible
- Source: entries/2026/05/29/topic-snapshot-disk-persistence.md

### [ACCEPT] es-snapshots-are-in-memory-dicts
Event sourcing snapshots on `store._snapshots` are plain Python dicts monkey-patched onto the store instance with no serialization; `getattr(self._store, '_snapshots', {})` defensively handles their absence
- Source: entries/2026/05/29/topic-snapshot-persistence.md

### [ACCEPT] snapshot-loss-triggers-full-replay
When event sourcing snapshots are missing, the system falls back to replaying all events from the log to reconstruct aggregate state — correct but O(n) in total event count
- Source: entries/2026/05/29/topic-snapshot-persistence.md

### [REJECT] mvcc-state-entirely-volatile
Duplicate of `mvcc-no-persistence` proposed above — same claim about MVCCDatabase having no persistence
- Source: entries/2026/05/29/topic-snapshot-persistence.md

### [REJECT] no-persistence-api-exists
Too broad — a grep returning zero matches for persistence keywords across the entire project is an observation about the current codebase state, not a durable invariant; also overlaps with `mvcc-no-persistence` and `es-snapshots-are-in-memory-dicts`
- Source: entries/2026/05/29/topic-snapshot-persistence.md

### [ACCEPT] mvcc-active-set-frozen-at-begin
`MVCCDatabase.begin_transaction` captures `active_at_start` as a frozen set of all currently active transaction IDs at that moment; it is never modified after creation and serves as the transaction's immutable snapshot boundary
- Source: entries/2026/05/29/topic-snapshot-vs-ssi-tradeoffs.md

### [ACCEPT] ssi-store-contains-only-committed-data
`SSIDatabase._store` only contains versions stamped with commit timestamps; uncommitted writes live in `tx._writes` until commit, so every version in the store is guaranteed to be from a committed transaction
- Source: entries/2026/05/29/topic-snapshot-vs-ssi-tradeoffs.md

### [REJECT] mvcc-visibility-requires-three-conditions
Duplicate of existing `visibility-requires-three-conditions`
- Source: entries/2026/05/29/topic-snapshot-vs-ssi-tradeoffs.md

### [ACCEPT] ssi-visibility-is-timestamp-comparison
`SSIDatabase._visible_value` determines visibility by a single comparison (`commit_ts <= snapshot_ts`) with no active-set tracking, because the deferred-write model guarantees all stored versions are committed
- Source: entries/2026/05/29/topic-snapshot-vs-ssi-tradeoffs.md

### [ACCEPT] ssi-own-writes-require-buffer-check
`SSIDatabase.read` must check `tx._writes` and `tx._deletes` before consulting `_store`, since a transaction's own writes aren't materialized into the shared store until commit — unlike MVCCDatabase where eager writes make own-writes visible automatically
- Source: entries/2026/05/29/topic-snapshot-vs-ssi-tradeoffs.md

### [ACCEPT] btree-split-write-order-protects-chain
During a leaf split, `_insert` writes the new right page to the WAL before the modified left page, ensuring the sibling pointer target is durable before any pointer references it — so the chain never points to a nonexistent page after crash recovery
- Source: entries/2026/05/29/topic-split-and-sibling-update-atomicity.md

### [REJECT] btree-wal-no-commit-record
Duplicate of existing `btree-wal-replays-without-commit` which captures the same structural fact that the B-tree WAL has no commit marker and replays all valid entries unconditionally
- Source: entries/2026/05/29/topic-split-and-sibling-update-atomicity.md

### [ACCEPT] btree-partial-split-replay-orphans-right-page
If a crash occurs after the right page and left page are WAL-logged but before the parent update, recovery produces a consistent sibling chain but the right page is unreachable from tree descent — range scans find the keys but point lookups silently miss them
- Source: entries/2026/05/29/topic-split-and-sibling-update-atomicity.md


## Entry: `topic-split-during-insert.md`

### [ACCEPT] btree-sibling-chain-fixed-offset
The `next_sibling` field sits at a fixed byte offset (bytes 3–6) in every serialized leaf page, making the sibling chain walkable without fully deserializing page contents
- Source: entries/2026/05/29/topic-split-during-insert.md

### [ACCEPT] iter-descends-left-spine
`__iter__` reaches the leftmost leaf by following `children[0]` at every internal level, then walks the sibling chain to `NO_SIBLING` to enumerate all keys in sorted order
- Source: entries/2026/05/29/topic-split-during-insert.md

### [REJECT] btree-split-inherits-sibling
Duplicate of `leaf-split-updates-sibling-pointers`
- Source: entries/2026/05/29/topic-split-during-insert.md

### [REJECT] btree-chain-no-cycle-detection
Duplicate of `range-scan-no-cycle-guard`
- Source: entries/2026/05/29/topic-split-during-insert.md

### [REJECT] btree-delete-patches-prev-sibling
Duplicate of `btree-leaf-sibling-patched-on-remove`
- Source: entries/2026/05/29/topic-split-during-insert.md

### [REJECT] btree-split-wal-gap
Duplicate of `btree-allocate-page-outside-wal`
- Source: entries/2026/05/29/topic-split-during-insert.md

---

## Entry: `topic-split-propagation.md`

### [REJECT] btree-splits-propagate-via-return-value
Duplicate of `btree-recursive-insert-returns-union`
- Source: entries/2026/05/29/topic-split-propagation.md

### [REJECT] btree-root-split-grows-height
Duplicate of `tree-height-monotonically-increases`
- Source: entries/2026/05/29/topic-split-propagation.md

### [REJECT] btree-leaf-copies-internal-promotes
Duplicate of `btree-leaf-split-copies-key`
- Source: entries/2026/05/29/topic-split-propagation.md

### [REJECT] btree-sibling-chain-maintained
Duplicate of `leaf-split-updates-sibling-pointers`
- Source: entries/2026/05/29/topic-split-propagation.md

---

## Entry: `topic-sqlite-durability-model.md`

### [ACCEPT] macos-fsync-incomplete-without-fullfsync
On macOS/Darwin, `os.fsync()` does not guarantee flushing the disk write cache to stable storage; only `fcntl(fd, F_FULLFSYNC)` provides that guarantee, meaning all 13 fsync sites in the codebase may provide no power-loss durability on the development platform
- Source: entries/2026/05/29/topic-sqlite-durability-model.md

### [REJECT] sqlite-fsyncs-directories-after-file-ops
About SQLite's internal behavior, not a claim about this codebase
- Source: entries/2026/05/29/topic-sqlite-durability-model.md

### [REJECT] sqlite-fcntl-sync-abstracts-platform-differences
About SQLite's VFS abstraction, not a claim about this codebase
- Source: entries/2026/05/29/topic-sqlite-durability-model.md

### [REJECT] lsm-flush-then-truncate-is-compound-failure
Duplicate of `lsm-flush-double-loss-window`
- Source: entries/2026/05/29/topic-sqlite-durability-model.md

### [REJECT] atomic-rename-is-not-durable-rename
Duplicate of `rename-without-dir-barrier`
- Source: entries/2026/05/29/topic-sqlite-durability-model.md

---

## Entry: `topic-sstable-footer-integrity.md`

### [ACCEPT] sstable-trailer-single-point-of-failure
The SSTable's 12-byte trailer (`footer_start` offset + entry `count`) is an unprotected single point of failure; its corruption makes the entire file's sparse index and data entries unreachable with no fallback discovery mechanism
- Source: entries/2026/05/29/topic-sstable-footer-integrity.md

### [ACCEPT] sparse-index-limits-corruption-blast-radius
A corrupted data entry in an SSTable only affects lookups within one sparse index segment (between two adjacent index offsets, default 16 entries); keys indexed by other segments remain correctly accessible
- Source: entries/2026/05/29/topic-sstable-footer-integrity.md

### [ACCEPT] sstable-short-read-guards-prevent-overrun
`_read_entry()` in the LSM SSTable checks for short reads on key and value fields, preventing buffer overruns on truncated files but silently skipping all remaining entries in the scan range
- Source: entries/2026/05/29/topic-sstable-footer-integrity.md

### [REJECT] sstable-no-integrity-checks
Duplicate of `lsm-and-sstable-have-no-checksums`
- Source: entries/2026/05/29/topic-sstable-footer-integrity.md

---

## Entry: `topic-sstable-magic-number-vs-crc.md`

### [ACCEPT] sstable-magic-is-only-integrity-check
The 4-byte magic `b"SSTB"` at offset 0 is the only data integrity check in the `sstable-and-compaction` SSTable read path; it gates file-type validation but provides no corruption detection for keys, values, timestamps, or structural offsets
- Source: entries/2026/05/29/topic-sstable-magic-number-vs-crc.md

### [ACCEPT] tombstone-marker-single-byte-no-redundancy
The tombstone/value distinction in `sstable-and-compaction/sstable.py` rests on a single byte (`0xFF` vs first byte of a 4-byte value length), making it vulnerable to single-bit corruption that silently converts live entries to deletions or vice versa
- Source: entries/2026/05/29/topic-sstable-magic-number-vs-crc.md

### [ACCEPT] entry-count-header-never-verified
The entry count stored in the SSTable header by `SSTableWriter.finish()` is trusted by `SSTableReader` but never validated against the actual number of entries read from the data section
- Source: entries/2026/05/29/topic-sstable-magic-number-vs-crc.md

### [REJECT] corrupted-key-length-cascades-silently
Duplicate of `header-corruption-derails-sequential-scan`
- Source: entries/2026/05/29/topic-sstable-magic-number-vs-crc.md


### [REJECT] stcs-compaction-is-count-triggered
Duplicate of existing `compaction-triggered-by-sstable-count`
- Source: entries/2026/05/29/topic-stcs-vs-lcs-tradeoffs.md

### [ACCEPT] lcs-levels-grow-exponentially
Leveled compaction sizes each level as `level_base_size * fanout^(level-1)` with a 10MB default base and 7 max levels, so each level is 10x the previous by default
- Source: entries/2026/05/29/topic-stcs-vs-lcs-tradeoffs.md

### [ACCEPT] lcs-l0-triggers-on-count-not-size
Level 0 in leveled compaction triggers based on SSTable count (`l0_compaction_trigger`), not total size, because L0 SSTables can have overlapping key ranges unlike higher levels which maintain non-overlapping invariants
- Source: entries/2026/05/29/topic-stcs-vs-lcs-tradeoffs.md

### [REJECT] newest-sstable-wins-reads
Duplicate of existing `lsm-newest-first-read-path` which already captures that get() searches newest-first and the first match is authoritative
- Source: entries/2026/05/29/topic-stcs-vs-lcs-tradeoffs.md

### [REJECT] compaction-removes-tombstones
Duplicate of existing `compact-purges-tombstones`, `sstable-tombstone-encoding-0xff`, and `bitcask-tombstone-is-empty-string` which collectively cover tombstone encoding and compaction-time removal
- Source: entries/2026/05/29/topic-stcs-vs-lcs-tradeoffs.md

### [ACCEPT] lsm-sstables-unprotected-mutation
`LSMTree._sstables` is mutated by both `_flush()` (append) and `compact()` (full replacement) with no synchronization, versioning, or ref counting, making concurrent reads unsafe if threading or async is added
- Source: entries/2026/05/29/topic-superversion-refcount-implementation.md

### [ACCEPT] compaction-deletes-before-reader-release
`compact()` deletes old SSTable files immediately after replacing `self._sstables`, with no mechanism to defer deletion until active readers holding references to the old list finish their iterations
- Source: entries/2026/05/29/topic-superversion-refcount-implementation.md

### [ACCEPT] sstable-compaction-manager-copies-list
`CompactionManager.get_sstables()` returns `list(self._sstables)` (a shallow copy), preventing iterator invalidation from concurrent mutation but not preventing file-deletion races on the underlying SSTable files
- Source: entries/2026/05/29/topic-superversion-refcount-implementation.md

### [ACCEPT] wal-module-uses-locking-lsm-does-not
The WAL module in `write-ahead-log/wal.py` uses `self._lock` for thread safety across its mutation paths, while the LSM tree module has zero locking or concurrency control despite having the same concurrent-access risks
- Source: entries/2026/05/29/topic-superversion-refcount-implementation.md

### [ACCEPT] gossip-suspicion-is-timeout-based
Suspicion in the gossip protocol is triggered by elapsed time exceeding `t_suspect` since the last heartbeat update, not by failed probe responses as in the full SWIM paper's ping/indirect-ping protocol
- Source: entries/2026/05/29/topic-swim-protocol.md

### [ACCEPT] gossip-death-is-irreversible-via-gossip
Once a node's status is `dead`, receiving a gossip message with `status: alive` for that node will never revert it — the merge logic at line 67 checks `local["status"] != "dead"` before allowing exoneration
- Source: entries/2026/05/29/topic-swim-protocol.md

### [ACCEPT] gossip-uses-full-membership-exchange
Each gossip round transmits the entire membership list via `deepcopy` rather than deltas or piggybacked updates, making per-exchange bandwidth cost O(N) in cluster size instead of O(1) amortized
- Source: entries/2026/05/29/topic-swim-protocol.md

### [ACCEPT] gossip-cleanup-bounds-membership-growth
Dead nodes are removed from the membership list after `t_cleanup` elapsed time (default 20), preventing unbounded growth of the membership table from accumulated failure records
- Source: entries/2026/05/29/topic-swim-protocol.md

### [ACCEPT] do-sync-force-bypasses-sync-mode
`_do_sync(force=True)` always calls fsync regardless of the configured `sync_mode`, ensuring durability-critical operations like segment rotation and WAL close are never silently skipped
- Source: entries/2026/05/29/topic-sync-mode-none-safety.md

### [ACCEPT] sync-mode-none-skips-per-write-fsync
When `sync_mode="none"`, the per-write `_do_sync()` call (default `force=False`) is a complete no-op — no fsync and no batch queue — trading durability for maximum write throughput
- Source: entries/2026/05/29/topic-sync-mode-none-safety.md

### [ACCEPT] wal-default-sync-mode-is-sync
`WriteAheadLog.__init__` defaults `sync_mode` to `"sync"`, making per-write fsync the safe default that callers must explicitly opt out of for higher throughput
- Source: entries/2026/05/29/topic-sync-mode-none-safety.md

### [ACCEPT] force-true-used-at-rotation-and-close
The only `_do_sync(force=True)` call sites (lines 165 and 175) correspond to segment rotation and WAL close, both operations where proceeding without a flush would risk data loss or file corruption
- Source: entries/2026/05/29/topic-sync-mode-none-safety.md

### [ACCEPT] tester-dev-suites-overlap-not-hierarchical
The tester and developer test suites have overlapping but non-hierarchical coverage: tester files uniquely test spec-example compliance and cross-path equivalence, while dev files uniquely test crash recovery and internal state
- Source: entries/2026/05/29/topic-test-coverage-gaps.md

### [ACCEPT] tester-spec-compliance-is-unique
`TestSpecExample` tests in tester files replay the exact spec usage example verbatim as a living executable spec, a form of spec-drift detection that developer test suites do not replicate
- Source: entries/2026/05/29/topic-test-coverage-gaps.md

### [ACCEPT] tester-files-never-import-internals
Tester test files import only the public API (e.g., `from btree import BTree`), while developer test files import internal types like `WAL`, `_serialize_leaf`, and `HEADER_FMT` for state injection and internal invariant checking
- Source: entries/2026/05/29/topic-test-coverage-gaps.md

### [REJECT] dev-suite-longer-but-not-superset
Duplicate: the length claim is already captured by `pytest-files-have-broader-coverage`, and the non-superset claim is captured by the proposed `tester-dev-suites-overlap-not-hierarchical`
- Source: entries/2026/05/29/topic-test-coverage-gaps.md


### [ACCEPT] tester-tests-not-auto-regenerated
`tester_test_*.py` files contain no auto-generation markers (`generated`, `DO NOT EDIT`) and no regeneration tooling exists in the documented code-expert workflow; they are static committed files
- Source: entries/2026/05/29/topic-tester-generation-pipeline.md

### [ACCEPT] tester-tests-are-distilled-specs
Each `tester_test_*.py` is a simplified subset of its corresponding `test_*.py`, organized by numbered behavioral properties (`# 1. No false negatives`, `# 2. FPR within 2x of target`) rather than exhaustive edge cases
- Source: entries/2026/05/29/topic-tester-generation-pipeline.md

### [REJECT] code-expert-has-no-test-generation-step
Describes the code-expert tool's workflow, not a code invariant of the target repository; tool behavior belongs in tool documentation, not the belief registry
- Source: entries/2026/05/29/topic-tester-generation-pipeline.md

### [REJECT] tester-and-original-tests-coexist
Implied by existing beliefs `pytest-files-have-broader-coverage`, `both-suites-discoverable-by-pytest`, and `tester-files-are-standalone-runnable`; no new invariant added
- Source: entries/2026/05/29/topic-tester-generation-pipeline.md

### [ACCEPT] tester-test-no-shared-harness
There is no shared `conftest.py`, base class, or test harness across `tester_test_*.py` files; each imports only its own module under test plus standard library utilities and is fully self-contained
- Source: entries/2026/05/29/topic-tester-test-file-locations.md

### [ACCEPT] tester-test-dual-runner
Some `tester_test_*.py` files are pytest-only (using fixtures/`pytest.raises`), while others include an `if __name__ == '__main__':` block for standalone execution; the two execution patterns coexist across modules
- Source: entries/2026/05/29/topic-tester-test-file-locations.md

### [REJECT] tester-test-195-functions
Unstable count detail — the number of test functions and classes changes as modules are added or tests are extended; not a durable invariant
- Source: entries/2026/05/29/topic-tester-test-file-locations.md

### [REJECT] tester-prefix-non-standard
Duplicates existing belief `tester-naming-avoids-pytest-discovery`
- Source: entries/2026/05/29/topic-tester-test-file-locations.md

### [REJECT] tester-test-files-colocated
Covered by existing belief `ddia-modules-self-contained`; the co-location pattern applies to all module artifacts including tester test files
- Source: entries/2026/05/29/topic-tester-test-file-locations.md

### [REJECT] tester-files-are-self-contained-runners
Duplicates existing belief `tester-files-are-standalone-runnable`
- Source: entries/2026/05/29/topic-tester-vs-pytest-test-duality.md

### [REJECT] test-files-cover-more-cases
Duplicates existing belief `pytest-files-have-broader-coverage`
- Source: entries/2026/05/29/topic-tester-vs-pytest-test-duality.md

### [REJECT] tester-bloom-filter-leaks-pytest
Duplicates existing belief `tester-pytest-boundary-is-leaky`
- Source: entries/2026/05/29/topic-tester-vs-pytest-test-duality.md

### [ACCEPT] fixture-usage-is-sparse
Only 3 of the `test_*.py` files use `@pytest.fixture` (`test_cdc`, `test_map_side_joins`, `test_bitcask`); most pytest files still use inline setup identical to their tester counterparts
- Source: entries/2026/05/29/topic-tester-vs-pytest-test-duality.md

### [ACCEPT] tester-print-includes-diagnostics
Tester files embed runtime metrics in their pass/fail output (tree height, page counts, pass/fail ratios) that pytest's structured reporting does not surface without the `-s` flag
- Source: entries/2026/05/29/topic-tester-vs-pytest-test-duality.md

### [ACCEPT] 2pc-participant-recover-returns-in-doubt-only
`Participant.recover()` identifies transactions stuck in `"prepared"` state but provides no mechanism to resolve them without the coordinator, making it a diagnostic tool rather than a recovery procedure
- Source: entries/2026/05/29/topic-three-phase-commit.md

### [ACCEPT] 2pc-blocking-window-is-between-prepare-and-decision
A participant that has voted YES in `prepare()` holds locks and cannot unilaterally commit or abort until it receives the coordinator's decision, creating an unbounded blocking window if the coordinator fails
- Source: entries/2026/05/29/topic-three-phase-commit.md

### [ACCEPT] 2pc-coordinator-recover-requires-liveness
`Coordinator.recover()` can only re-send decisions to participants that are currently available (`is_available()` check), leaving unavailable participants still in-doubt with no resolution path
- Source: entries/2026/05/29/topic-three-phase-commit.md

### [ACCEPT] 2pc-locks-held-during-in-doubt
Locks acquired during `prepare()` are only released by `commit()` or `abort()`, meaning an in-doubt transaction blocks all subsequent transactions on the same keys indefinitely until the coordinator recovers
- Source: entries/2026/05/29/topic-three-phase-commit.md

### [REJECT] lsht-tombstone-is-real-value
Covered by existing belief `log-structured-sentinel-is-in-band`; the structural identity with normal records is the defining characteristic of an in-band sentinel
- Source: entries/2026/05/29/topic-tombstone-encoding.md

### [REJECT] hash-index-tombstone-is-zero-length
Covered by existing belief `bitcask-tombstone-is-empty-string`; zero-length value is the direct encoding of empty string
- Source: entries/2026/05/29/topic-tombstone-encoding.md

### [REJECT] hint-files-cannot-represent-deletes
Covered by existing beliefs `lsht-hint-no-tombstone-flag` and `bitcask-hint-files-exclude-tombstones`, which describe this limitation for both implementations
- Source: entries/2026/05/29/topic-tombstone-encoding.md

### [ACCEPT] tombstone-sentinel-is-a-forbidden-value
Storing `b"__BITCASK_TOMBSTONE__"` as a legitimate value in `log-structured-hash-table` causes silent data loss on next recovery, as `_scan_segment` interprets it as a delete — the sentinel is an implicit API constraint with no validation
- Source: entries/2026/05/29/topic-tombstone-encoding.md


### [ACCEPT] distributed-tombstone-gc-requires-downtime-bound
Any tombstone garbage collection strategy must define a maximum tolerated node downtime; tombstones removed before a down node receives them cause data resurrection via read repair or anti-entropy.
- Source: entries/2026/05/29/topic-tombstone-gc-and-repair-window.md

### [REJECT] dynamo-anti-entropy-unions-all-keys
Duplicates existing belief `anti-entropy-unions-all-keys` which already captures that `anti_entropy_repair()` collects keys via set union across all available nodes.
- Source: entries/2026/05/29/topic-tombstone-gc-and-repair-window.md

### [REJECT] gc-grace-seconds-is-repair-window
Describes Cassandra's external design pattern rather than a testable claim about this codebase; useful context but not a codebase invariant.
- Source: entries/2026/05/29/topic-tombstone-gc-and-repair-window.md

### [ACCEPT] reference-implementations-lack-tombstone-ttl
None of the DDIA reference implementations combine tombstone support with time-bounded garbage collection; the multi-leader module stores tombstones indefinitely and the Dynamo module has no delete support at all.
- Source: entries/2026/05/29/topic-tombstone-gc-and-repair-window.md

### [REJECT] tombstone-lifetime-space-safety-tradeoff
General distributed systems design principle (longer tombstone lifetimes waste space but tolerate longer outages) rather than a testable claim about this codebase's behavior.
- Source: entries/2026/05/29/topic-tombstone-gc-and-repair-window.md

### [ACCEPT] gossip-topology-is-fully-connected
GossipCluster uses unrestricted random peer selection (any node can gossip with any other), making the effective topology a fully-connected graph.
- Source: entries/2026/05/29/topic-topology-propagation-bounds.md

### [ACCEPT] gossip-fanout-is-one
Each node selects exactly one random peer per gossip round, with bidirectional exchange producing at most N pairwise syncs per round.
- Source: entries/2026/05/29/topic-topology-propagation-bounds.md

### [ACCEPT] gossip-convergence-is-olog-n
The gossip implementation converges in O(log N) rounds, empirically validated up to N=64 with the bound `5*log2(N)+5` in `test_convergence_speed_olog_n`.
- Source: entries/2026/05/29/topic-topology-propagation-bounds.md

### [ACCEPT] no-structured-topology-support
The codebase has no ring, star, or mesh topology modes for gossip; peer selection is always uniform random from all active nodes.
- Source: entries/2026/05/29/topic-topology-propagation-bounds.md

### [ACCEPT] add-node-incremental-ring-mutation
`add_node` mutates the ring after each vnode insertion, so subsequent vnodes in the same loop see predecessors from earlier iterations, preventing overlapping transfer arcs.
- Source: entries/2026/05/29/topic-transfer-map-accuracy.md

### [ACCEPT] add-node-same-owner-guard
The `old_owner != node_id` guard at `consistent_hashing.py:38` skips transfer recording when a new vnode's successor is another vnode of the same node, avoiding double-counting already-claimed arcs.
- Source: entries/2026/05/29/topic-transfer-map-accuracy.md

### [ACCEPT] transfer-test-weak-assertions
`test_add_node_returns_transfers` only asserts transfer direction (A→B) and existence, not arc non-overlap or total size correctness — the code is correct but the test doesn't prove it.
- Source: entries/2026/05/29/topic-transfer-map-accuracy.md

### [ACCEPT] transfer-dict-keyed-by-arc
The transfer dict uses `(arc_start, arc_end)` tuples as keys, so two vnodes producing identical arc boundaries would silently overwrite rather than accumulate.
- Source: entries/2026/05/29/topic-transfer-map-accuracy.md

### [ACCEPT] vnode-150-guarantees-sub-1.5-imbalance
With 3 equally-weighted nodes and 150 vnodes each, `load_imbalance()` is asserted to stay below 1.5 (`test_consistent_hashing.py:49`).
- Source: entries/2026/05/29/topic-virtual-node-count-tuning.md

### [REJECT] vnode-count-scales-with-weight
Duplicates existing belief `ch-vnode-count-is-weighted` which already captures that a node's actual vnode count is `int(num_vnodes * weight)`.
- Source: entries/2026/05/29/topic-virtual-node-count-tuning.md

### [ACCEPT] add-node-mutation-cost-is-quadratic-in-ring-size
Each `add_node` call performs `vnode_count` list insertions into a sorted list, each O(total ring entries), making node addition O(V × N×V) in the worst case.
- Source: entries/2026/05/29/topic-virtual-node-count-tuning.md

### [ACCEPT] lookup-cost-is-logarithmic-in-total-vnodes
`get_node` performs a single `bisect.bisect` over `_ring_positions`, so key lookup is O(log(N×V)) regardless of cluster size.
- Source: entries/2026/05/29/topic-virtual-node-count-tuning.md

### [ACCEPT] more-vnodes-monotonically-improves-balance
The test `test_vnode_count_affects_balance` asserts that imbalance at 500 vnodes is strictly less than at 1 vnode, and the statistical model (variance ∝ 1/V) guarantees monotonic improvement.
- Source: entries/2026/05/29/topic-virtual-node-count-tuning.md

### [REJECT] lsm-wal-truncation-is-total
Covered by existing beliefs `lsm-wal-truncate-no-arg` (takes no argument) and `lsm-wal-truncate-destroys-immediately` (erases immediately), which together establish total truncation semantics.
- Source: entries/2026/05/29/topic-wal-checkpoint-protocol.md

### [REJECT] wal-checkpoint-is-explicit-record-type
Already implied by existing beliefs `checkpoint-record-not-used-in-recovery`, `wal-checkpoint-consumes-sequence`, and `checkpoint-filtered-from-replay`, which collectively establish that OP_CHECKPOINT is a record type in the WAL stream.
- Source: entries/2026/05/29/topic-wal-checkpoint-protocol.md

### [ACCEPT] memtable-threshold-bounds-recovery
The `memtable_threshold` parameter (`lsm.py:202`) directly caps the maximum number of WAL entries that must be replayed on crash recovery, because the WAL is truncated on every flush.
- Source: entries/2026/05/29/topic-wal-checkpoint-protocol.md

### [ACCEPT] wal-truncate-preserves-records-above-seq
`WriteAheadLog.truncate(up_to_seq)` keeps records with `seq_num > up_to_seq` and deletes only those at or below, enabling partial log reclamation tied to checkpoint boundaries.
- Source: entries/2026/05/29/topic-wal-checkpoint-protocol.md


Looking at the 5 entries and cross-referencing against 150+ existing beliefs.

---

## Entry 1: `topic-wal-checksum-framing.md`

### [REJECT] wal-has-crc32-validation
Covered by `wal-crc-includes-op-type` (CRC computation) and `wal-truncation-vs-corruption-distinction` (error behavior on mismatch)
- Source: entries/2026/05/29/topic-wal-checksum-framing.md

### [REJECT] wal-crc-excludes-seq-num
Duplicate of `wal-crc-does-not-cover-seqnum` and `seq-num-corruption-is-silent`
- Source: entries/2026/05/29/topic-wal-checksum-framing.md

### [REJECT] wal-corruption-discards-tail
Covered by `wal-corruption-stops-per-file` (stops at first corruption per file, continues to next) and `iterate-stops-silently-at-corruption`
- Source: entries/2026/05/29/topic-wal-checksum-framing.md

### [REJECT] wal-length-corruption-mimics-truncation
Covered by `torn-length-prefix-causes-silent-skip` (garbage length consumes valid records, returns None) and `wal-partial-read-is-eof` (short reads treated as EOF)
- Source: entries/2026/05/29/topic-wal-checksum-framing.md

---

## Entry 2: `topic-wal-commit-protocol.md`

### [ACCEPT] wal-no-commit-method
The standalone `WriteAheadLog` class in `write-ahead-log/wal.py` has no `commit` method; sync-then-truncate coordination must be implemented by a higher-level storage engine that composes the WAL with a data file
- Source: entries/2026/05/29/topic-wal-commit-protocol.md

### [REJECT] truncate-fsyncs-before-rewrite
Duplicate of `wal-truncate-fsyncs-before-scan`
- Source: entries/2026/05/29/topic-wal-commit-protocol.md

### [ACCEPT] wal-not-imported-outside-tests
No production module imports the standalone `WriteAheadLog`; it is only referenced in `test_wal.py` and `tester_test_wal.py`, meaning crash-safe commit coordination using this module is unimplemented
- Source: entries/2026/05/29/topic-wal-commit-protocol.md

### [REJECT] rotate-missing-directory-fsync
Duplicate of `wal-no-directory-fsync` which already covers all segment creation and deletion
- Source: entries/2026/05/29/topic-wal-commit-protocol.md

---

## Entry 3: `topic-wal-commit-vs-metadata-sync-ordering.md`

### [ACCEPT] wal-commit-syncs-metadata-implicitly
`WAL.commit()` in the B-tree engine syncs metadata page 0 because `PageManager.sync()` fsyncs the single shared file descriptor that holds both data pages and metadata; there is no dedicated metadata fsync step
- Source: entries/2026/05/29/topic-wal-commit-vs-metadata-sync-ordering.md

### [REJECT] commit-fence-ordering
Duplicate of `btree-commit-syncs-data-before-wal-truncate`
- Source: entries/2026/05/29/topic-wal-commit-vs-metadata-sync-ordering.md

### [ACCEPT] single-file-design-enables-atomic-sync
`PageManager` stores metadata (page 0) and data pages in a single file, so one `os.fsync()` call in `sync()` covers both; splitting into separate files would require multiple fsyncs with ordering constraints in the commit fence
- Source: entries/2026/05/29/topic-wal-commit-vs-metadata-sync-ordering.md

---

## Entry 4: `topic-wal-crash-safety-gap.md`

### [ACCEPT] wal-truncate-requires-prior-checkpoint
WAL truncation is only safe because it is called after the data the WAL protects has been durably written to the main store (SSTable flush or data file fsync); violating this ordering loses committed data with no recovery path
- Source: entries/2026/05/29/topic-wal-crash-safety-gap.md

### [REJECT] lsm-wal-truncate-not-atomic
Covered by `lsm-wal-truncate-destroys-immediately` (wb mode zeroes content with no intermediate durable state) and `lsm-wal-truncate-wb-then-ab` (the two-phase reopen pattern)
- Source: entries/2026/05/29/topic-wal-crash-safety-gap.md

### [REJECT] wal-truncate-holds-lock
Duplicate of `wal-truncate-blocks-all-operations`
- Source: entries/2026/05/29/topic-wal-crash-safety-gap.md

### [ACCEPT] wal-partial-truncate-idempotent-replay
If the standalone WAL's `truncate()` crashes mid-iteration over files, un-processed files retain old records that will be replayed on recovery; this is safe because PUT and DELETE are idempotent against an already-current store
- Source: entries/2026/05/29/topic-wal-crash-safety-gap.md

### [REJECT] wal-fsync-before-close-on-truncate
Duplicate of `wal-truncate-fsyncs-before-scan`
- Source: entries/2026/05/29/topic-wal-crash-safety-gap.md

---

## Entry 5: `topic-wal-crash-safety-ordering.md`

### [REJECT] btree-wal-fsync-before-data-write
Duplicate of `btree-wal-before-data` and `btree-wal-fsync-per-entry`
- Source: entries/2026/05/29/topic-wal-crash-safety-ordering.md

### [REJECT] btree-commit-syncs-data-before-wal-truncate
Already exists as `btree-commit-syncs-data-before-wal-truncate`
- Source: entries/2026/05/29/topic-wal-crash-safety-ordering.md

### [REJECT] lsm-wal-no-fsync
Duplicate of `lsm-wal-has-no-fsync`
- Source: entries/2026/05/29/topic-wal-crash-safety-ordering.md

### [REJECT] btree-redo-is-idempotent
Duplicate of `btree-wal-replay-is-idempotent` and `btree-wal-is-physical-redo-only`
- Source: entries/2026/05/29/topic-wal-crash-safety-ordering.md

### [REJECT] btree-recovery-is-crash-safe
Duplicate of `wal-recover-then-truncate` which states recovery fsyncs the data file before truncating the WAL, so a crash during recovery leaves the WAL intact for re-replay
- Source: entries/2026/05/29/topic-wal-crash-safety-ordering.md

---

**Summary:** 6 accepted, 14 rejected (all duplicates of existing beliefs). The new beliefs capture: the standalone WAL's lack of commit coordination (2), the B-tree's implicit metadata sync through single-file design (2), and two cross-cutting invariants about truncation safety and partial-truncation idempotency.


### [ACCEPT] lsm-wal-has-no-commit-markers
The LSM WAL (`lsm.py:13-65`) appends bare key-value pairs with no commit or transaction boundaries, safe only because structural changes (compaction, SSTable creation) use atomic file operations outside the WAL
- Source: entries/2026/05/29/topic-wal-operation-boundaries.md

### [ACCEPT] wal-open-latest-init-only
`_open_latest` is called exclusively from `__init__`, so the TOCTOU window between `_wal_files()` and `os.path.getsize()` cannot be hit by concurrent WAL operations
- Source: entries/2026/05/29/topic-wal-size-check-toctou.md

### [ACCEPT] wal-single-writer-thread-level
The WAL enforces single-writer via `threading.Lock` but has no inter-process locking mechanism (no flock/PID file), so the single-writer invariant holds only within a single OS process
- Source: entries/2026/05/29/topic-wal-size-check-toctou.md

### [ACCEPT] wal-hot-path-avoids-getsize
`_maybe_rotate` uses `self._fd.tell()` instead of `os.path.getsize()`, avoiding filesystem stat calls and TOCTOU races on every append
- Source: entries/2026/05/29/topic-wal-size-check-toctou.md

### [ACCEPT] wal-all-mutations-under-lock
Every WAL method that writes records or modifies files (`append`, `append_batch`, `checkpoint`, `truncate`) acquires `self._lock` before performing any I/O
- Source: entries/2026/05/29/topic-wal-size-check-toctou.md

### [ACCEPT] no-shadow-paging-implementation-exists
The codebase contains no copy-on-write or shadow paging implementation; all three storage engines use write-ahead logging exclusively for crash recovery
- Source: entries/2026/05/29/topic-wal-vs-shadow-paging.md

### [REJECT] wal-fsync-in-sync-and-batch-modes
Covered by existing `wal-do-sync-branches-mutually-exclusive`, `batch-mode-defers-fsync`, and `none-mode-never-syncs-on-append`
- Source: entries/2026/05/29/topic-wal-fsync-durability.md

### [REJECT] btree-selective-fsync
Covered by combination of `btree-wal-fsync-per-entry` (WAL fsyncs) and `write-meta-no-fsync` (PageManager metadata does not)
- Source: entries/2026/05/29/topic-wal-fsync-durability.md

### [REJECT] wal-sync-mode-none-is-unsafe
Durability gap covered by `none-mode-never-syncs-on-append`; testing observation is not a code invariant
- Source: entries/2026/05/29/topic-wal-fsync-durability.md

### [REJECT] btree-wal-logs-full-page-images
Already exists as `btree-wal-logs-full-page-images`
- Source: entries/2026/05/29/topic-wal-operation-boundaries.md

### [REJECT] wal-commit-record-is-atomicity-boundary
Covered by existing `wal-batch-atomicity-via-commit-record` and `wal-batch-commit-sentinel`
- Source: entries/2026/05/29/topic-wal-operation-boundaries.md

### [REJECT] btree-wal-has-no-multi-step-grouping
Covered by existing `btree-wal-no-operation-boundaries`
- Source: entries/2026/05/29/topic-wal-operation-boundaries.md

### [REJECT] wal-replay-flat-filter
Covered by existing `wal-replay-ignores-commit`, `replay-does-not-enforce-batch-atomicity`, and `wal-replay-no-atomicity-check`
- Source: entries/2026/05/29/topic-wal-recovery-consumer.md

### [REJECT] wal-commit-is-metadata-only
Covered by existing `wal-docstring-describes-intent-not-behavior` and `replay-does-not-enforce-batch-atomicity`
- Source: entries/2026/05/29/topic-wal-recovery-consumer.md

### [REJECT] wal-iterate-vs-replay
Covered by existing `wal-iterate-preserves-all-ops` and `checkpoint-filtered-from-replay`
- Source: entries/2026/05/29/topic-wal-recovery-consumer.md

### [REJECT] wal-no-partial-batch-test
Close to existing `wal-no-torn-write-tests`; partial batch is a subset of torn write scenarios
- Source: entries/2026/05/29/topic-wal-recovery-consumer.md

### [REJECT] wal-stops-replay-on-crc-failure
Already exists as `wal-crc-mismatch-halts-all-replay`
- Source: entries/2026/05/29/topic-wal-vs-shadow-paging.md

### [REJECT] btree-wal-commit-requires-data-fsync-before-truncate
Covered by existing `wal-commit-sync-before-truncate` and `btree-wal-truncate-is-safe`
- Source: entries/2026/05/29/topic-wal-vs-shadow-paging.md


### [REJECT] watermark-monotonic-advance
Duplicate of existing `stream-join-watermark-monotonic`
- Source: entries/2026/05/29/topic-watermark-vs-processing-time.md

### [ACCEPT] late-event-drop-is-hard-cutoff
Events with `timestamp < watermark - allowed_lateness` are unconditionally dropped and counted in `stats.late_events_dropped`; there is no secondary path to recover or re-buffer them
- Source: entries/2026/05/29/topic-watermark-vs-processing-time.md

### [REJECT] miss-emission-requires-expiration
Duplicate of existing `stream-join-miss-deferred`
- Source: entries/2026/05/29/topic-watermark-vs-processing-time.md

### [ACCEPT] join-determinism-from-event-time
The stream join processor's output is fully determined by the sequence of `(stream_name, key, value, timestamp)` inputs and `advance_time` calls, with no dependency on wall-clock time — making it deterministically testable without clock mocking
- Source: entries/2026/05/29/topic-watermark-vs-processing-time.md

### [ACCEPT] buffer-bounded-by-window-plus-lateness
Join buffer size is bounded by events within `window.duration` of the current watermark; `_expire_events` garbage-collects everything below that cutoff on every event arrival or `advance_time` call
- Source: entries/2026/05/29/topic-watermark-vs-processing-time.md

### [ACCEPT] advance-time-injects-external-watermark
`advance_time(timestamp)` lets an external orchestrator push the watermark forward without receiving an actual event, enabling idle-stream handling and explicit trigger semantics; it rejects timestamps at or below the current watermark
- Source: entries/2026/05/29/topic-watermark-vs-processing-time.md

### [ACCEPT] lsm-no-write-counters
The LSM-tree engine has no built-in byte or operation counters for writes; all write instrumentation (WAL appends, SSTable flushes, compaction output) must be added from scratch
- Source: entries/2026/05/29/topic-write-amplification-measurement.md

### [ACCEPT] btree-has-page-io-counters
The B-tree's `PageManager` tracks `pages_read` and `pages_written` with a `reset_counters()` method (multiply by `page_size` for bytes), but WAL writes are not counted by any existing counter
- Source: entries/2026/05/29/topic-write-amplification-measurement.md

### [REJECT] btree-page-write-always-full
Duplicate of existing `page-padding-to-page-size`
- Source: entries/2026/05/29/topic-write-amplification-measurement.md

### [ACCEPT] lsm-wal-entry-size
Each LSM WAL entry is exactly `8 + len(key_utf8) + len(value)` bytes: two 4-byte big-endian length headers plus the raw payloads, with no checksum or operation-type field
- Source: entries/2026/05/29/topic-write-amplification-measurement.md

### [ACCEPT] btree-wal-entry-includes-full-page
Each B-tree WAL entry is `16 + page_size` bytes (12-byte header with seq/page_num/data_len, full page image, 4-byte CRC32), making WAL writes the dominant cost for small key-value updates
- Source: entries/2026/05/29/topic-write-amplification-measurement.md

### [REJECT] lsm-wal-missing-fsync
Duplicate of existing `lsm-wal-has-no-fsync`
- Source: entries/2026/05/29/topic-write-ordering-and-barriers.md

### [REJECT] codebase-uses-fsync-not-fdatasync
Duplicate of existing `fdatasync-never-used`
- Source: entries/2026/05/29/topic-write-ordering-and-barriers.md

### [REJECT] btree-wal-fsync-before-data-write
Duplicate of existing `btree-wal-before-data`
- Source: entries/2026/05/29/topic-write-ordering-and-barriers.md

### [REJECT] no-directory-fsync-after-file-creation
Duplicate of existing `wal-no-directory-fsync`
- Source: entries/2026/05/29/topic-write-ordering-and-barriers.md

### [ACCEPT] tob-quorum-is-strict-majority
The TOB cluster uses 3 nodes where 2 can make progress but 1 alone cannot, confirming a strict-majority quorum requirement for both Paxos Phase 1 and Phase 2
- Source: entries/2026/05/29/total-order-broadcast-test_total_order_broadcast.md

### [ACCEPT] tob-consensus-uses-paxos-ballot-numbers
`ConsensusInstance.prepare(n)` / `accept(n, val)` use monotonic proposal numbers where a higher prepare preempts a lower one, causing `accept` with a stale number to return `{accepted: False}`
- Source: entries/2026/05/29/total-order-broadcast-test_total_order_broadcast.md

### [ACCEPT] tob-linearizable-register-is-built-on-broadcast
`LinearizableRegister` wraps `TOBCluster` to provide `write()`/`read()`/`compare_and_set()`, demonstrating the DDIA Chapter 9 equivalence between total-order broadcast and linearizable storage
- Source: entries/2026/05/29/total-order-broadcast-test_total_order_broadcast.md

### [ACCEPT] tob-recovery-replays-missed-slots
After `recover_node()`, the recovered node's delivery order matches live nodes' order, meaning recovery replays all consensus decisions made while the node was down via state transfer
- Source: entries/2026/05/29/total-order-broadcast-test_total_order_broadcast.md

### [ACCEPT] tob-paxos-per-slot
Each slot in the total order log is decided by an independent single-decree Paxos instance (`ConsensusInstance`); there is no stable-leader optimization or Multi-Paxos Phase 1 skip
- Source: entries/2026/05/29/total-order-broadcast-total_order_broadcast.md

### [ACCEPT] tob-contiguous-delivery
Messages are delivered to the application only when all prior slots are decided; `_next_slot` advances through a contiguous run, and a gap (undecided slot N when N+1 is decided) blocks all later delivery
- Source: entries/2026/05/29/total-order-broadcast-total_order_broadcast.md

### [ACCEPT] tob-proposal-number-encodes-node-id
Proposal numbers use `round * num_nodes + node_id` for global uniqueness without coordination — node 0's proposals are 0, 3, 6, ... and node 1's are 1, 4, 7, ... in a 3-node cluster
- Source: entries/2026/05/29/total-order-broadcast-total_order_broadcast.md

### [ACCEPT] tob-value-adoption-on-prepare
In `_handle_prepare_response`, if any acceptor reports a previously accepted value, the proposer must adopt the one with the highest proposal number — this is the core Paxos safety invariant preventing decided values from being overwritten
- Source: entries/2026/05/29/total-order-broadcast-total_order_broadcast.md

### [ACCEPT] tob-preempted-value-requeued
When a proposer's value loses its slot to a competing value, the original is pushed back onto `_pending` for proposal in a later slot, ensuring no broadcast messages are silently dropped
- Source: entries/2026/05/29/total-order-broadcast-total_order_broadcast.md

### [ACCEPT] tob-linearizable-reads-via-broadcast
`LinearizableRegister.read()` broadcasts the read through consensus rather than reading locally, establishing a consistent point in the total order — necessary for linearizability without leader leases
- Source: entries/2026/05/29/total-order-broadcast-total_order_broadcast.md

### [ACCEPT] tob-no-persistent-storage
TOBNode crash/recovery uses state transfer from peers via `force_decide()` on each missed slot; there is no WAL, persistent log, or on-disk state — all durability depends on a majority staying alive
- Source: entries/2026/05/29/total-order-broadcast-total_order_broadcast.md


### [ACCEPT] 2pc-abort-guarantees-no-side-effects
When any participant is unavailable, `execute()` returns `"aborted"` and no participant's state is modified, even for participants that were available and could have committed independently
- Source: entries/2026/05/29/two-phase-commit-test_2pc.md

### [ACCEPT] 2pc-uses-key-level-locking
Participants lock at key granularity during `prepare()`; a second transaction touching a locked key receives a `"no"` vote and aborts rather than waiting or queuing
- Source: entries/2026/05/29/two-phase-commit-test_2pc.md

### [ACCEPT] 2pc-locks-released-on-both-paths
Locks held by a transaction are released regardless of whether the transaction commits or aborts, preventing deadlocks across sequential transactions
- Source: entries/2026/05/29/two-phase-commit-test_2pc.md

### [ACCEPT] 2pc-coordinator-recovery-replays-commit
A coordinator with a `"committing"` log entry will re-send commit decisions to all participants during `recover()`, ensuring that a crash after the commit decision but before delivery still completes the transaction
- Source: entries/2026/05/29/two-phase-commit-test_2pc.md

### [ACCEPT] 2pc-result-dict-not-exceptions
The 2PC protocol communicates all outcomes (commits, aborts, lock conflicts, unavailability) via `{"outcome": "committed"|"aborted", "reason": ...}` return dicts; no method raises exceptions
- Source: entries/2026/05/29/two-phase-commit-test_2pc.md

### [ACCEPT] 2pc-unanimous-vote
A transaction commits only if every participant in `participant_operations` votes `"yes"`; a single `"no"` vote triggers abort for all participants regardless of how many voted yes
- Source: entries/2026/05/29/two-phase-commit-two_phase_commit.md

### [REJECT] 2pc-no-exception-errors
Duplicates `2pc-result-dict-not-exceptions` which captures the same claim with a more descriptive ID
- Source: entries/2026/05/29/two-phase-commit-two_phase_commit.md

### [ACCEPT] 2pc-timeout-unused
The `timeout` parameter is accepted by `Coordinator.__init__` but never referenced in any logic; availability is modeled via boolean `_available` flags on participants instead of actual time-based timeouts
- Source: entries/2026/05/29/two-phase-commit-two_phase_commit.md

### [ACCEPT] 2pc-log-before-decision
The coordinator appends `"committing"` or `"aborting"` to its log before broadcasting the decision to participants, enabling crash recovery via `recover()` to re-send decisions that were never delivered
- Source: entries/2026/05/29/two-phase-commit-two_phase_commit.md

### [ACCEPT] 2pc-lock-ownership-guard
`Participant.abort()` only releases locks where `self.locks.get(op["key"]) == tx_id`, preventing one transaction's abort from releasing another transaction's lock
- Source: entries/2026/05/29/two-phase-commit-two_phase_commit.md

### [ACCEPT] unbundled-catchup-rebuild-equivalence
Catch-up via `snapshot_and_stream` and rebuild via full CDC event replay must produce identical derived-system state, verified by comparing `get_state()` output from two independent derived systems
- Source: entries/2026/05/29/unbundled-database-test_tester_validation.md

### [ACCEPT] unbundled-lsn-sequential
LSNs returned by the unbundled database's `put()`/`delete()` are 1-indexed and strictly sequential with no gaps
- Source: entries/2026/05/29/unbundled-database-test_tester_validation.md

### [ACCEPT] unbundled-rebuild-clears-state
`StorageEngine.rebuild(wal)` clears all existing data (including manually injected entries) before replaying WAL entries, ensuring no phantom or stale state survives a rebuild
- Source: entries/2026/05/29/unbundled-database-test_tester_validation.md

### [ACCEPT] unbundled-flush-zeroes-lag
After `db.flush()`, `get_lag()` returns 0 for all derived systems, confirming all pending CDC events have been consumed
- Source: entries/2026/05/29/unbundled-database-test_tester_validation.md

### [REJECT] unbundled-cdc-update-carries-old-and-new
Duplicates existing belief `unbundled-db-cdc-events-carry-old-value`; the new_value presence is implicit in any CDC event representing a mutation
- Source: entries/2026/05/29/unbundled-database-test_tester_validation.md

### [ACCEPT] wal-is-source-of-truth
In the unbundled database, `StorageEngine` has no public mutation method other than `apply(WALEntry)` and `rebuild(wal)`; all state changes must flow through `WriteAheadLog.append()` first, making the WAL the authoritative record
- Source: entries/2026/05/29/unbundled-database-unbundled_database.md

### [ACCEPT] cdc-insert-update-distinction
Insert vs. update is determined solely by whether `CDCStream.emit()` receives a non-None `old_value`; the WAL itself only records `PUT` operations for both cases
- Source: entries/2026/05/29/unbundled-database-unbundled_database.md

### [REJECT] lazy-derived-propagation
Duplicates existing belief `unbundled-db-flush-required-for-derived` which captures the same decoupling between writes and derived-system propagation
- Source: entries/2026/05/29/unbundled-database-unbundled_database.md

### [ACCEPT] consumer-independent-positions
Each `DerivedSystem` tracks its own LSN position independently via `_position`, allowing consumers to fall behind or catch up at different rates without blocking each other
- Source: entries/2026/05/29/unbundled-database-unbundled_database.md

### [ACCEPT] derived-systems-rebuildable-from-cdc
Every `DerivedSystem` implements `rebuild(events)` that clears state and replays from scratch, guaranteeing eventual convergence with the CDC event log regardless of prior state
- Source: entries/2026/05/29/unbundled-database-unbundled_database.md

### [REJECT] gitignore-covers-python-only
Low value — easily verified by reading the 3-line `.gitignore` file directly
- Source: entries/2026/05/29/unknown.md

### [REJECT] gitignore-applies-repo-wide
Obvious from reading the file; unprefixed patterns matching at any depth is default gitignore behavior
- Source: entries/2026/05/29/unknown.md

### [ACCEPT] no-disk-artifact-patterns
On-disk files created by storage-engine modules (B-tree pages, WAL segments, SSTable files) are not covered by `.gitignore`, relying on test teardown or manual cleanup
- Source: entries/2026/05/29/unknown.md

### [REJECT] pyc-pattern-is-redundant-on-py3
Trivia about Python internals; doesn't affect how developers work in the codebase
- Source: entries/2026/05/29/unknown.md


### [ACCEPT] versioned-store-sibling-semantics
`VersionedKVStore.get` returns a list of `VersionedValue` entries; concurrent (causally unrelated) writes produce multiple siblings, and the store never auto-resolves conflicts — clients must call `reconcile`
- Source: entries/2026/05/29/vector-clocks-test_vector_clock.md

### [ACCEPT] context-parameter-establishes-causality
Passing a vector clock as `context` to `VersionedKVStore.put` declares the write descends from that version; omitting context treats the write as concurrent with all existing versions
- Source: entries/2026/05/29/vector-clocks-test_vector_clock.md

### [ACCEPT] find-conflicts-detects-concurrency
`find_conflicts` returns `True` if and only if at least two `VersionedValue` entries in the input list have concurrent (mutually non-dominating) vector clocks
- Source: entries/2026/05/29/vector-clocks-test_vector_clock.md

### [ACCEPT] vector-clock-compare-partial-order
`VectorClock.compare` returns one of four string values — `BEFORE`, `AFTER`, `EQUAL`, or `CONCURRENT` — implementing a partial order where the symmetric property holds (`BEFORE` ↔ `AFTER`)
- Source: entries/2026/05/29/vector-clocks-test_vector_clock.md

### [ACCEPT] vc-prune-keeps-highest-counters
`VectorClock.prune(n)` retains the `n` entries with the highest counter values and discards the rest, using `sorted` descending by counter
- Source: entries/2026/05/29/vector-clocks-vector_clock-prune.md

### [ACCEPT] vc-prune-is-lossy
Pruning permanently discards causal information; subsequent `compare()` or `dominates()` calls treat pruned nodes as counter=0, which can produce false concurrency or false dominance
- Source: entries/2026/05/29/vector-clocks-vector_clock-prune.md

### [ACCEPT] vc-prune-tiebreak-unstable
When multiple nodes share the same counter value during `prune`, which survive depends on Python's `sorted` stability over dict iteration order — not a deterministic tiebreak policy callers can rely on
- Source: entries/2026/05/29/vector-clocks-vector_clock-prune.md

### [ACCEPT] snapshot-returns-detached-copy
`SSIDatabase._snapshot` returns a new dict each call; callers in `read_predicate` and `commit` mutate it freely (overlaying writes, removing deletes) without corrupting the MVCC store
- Source: entries/2026/05/29/write-skew-detection-ssi_database-_snapshot.md

### [ACCEPT] snapshot-excludes-uncommitted-writes
`_snapshot` only includes committed versions (`commit_ts <= snapshot_ts`); all three call sites manually overlay `tx._writes` and `tx._deletes` when they need the transaction-local view
- Source: entries/2026/05/29/write-skew-detection-ssi_database-_snapshot.md

### [ACCEPT] snapshot-uses-next-timestamp-for-current-state
During commit validation, `_snapshot(self._next_timestamp)` captures all committed versions because `_next_timestamp` is strictly greater than any existing `commit_ts`
- Source: entries/2026/05/29/write-skew-detection-ssi_database-_snapshot.md

### [ACCEPT] read-predicate-stores-result-snapshot
`read_predicate` appends a `(predicate, dict(result))` tuple to `tx._predicate_locks` — a deep copy so `commit()` can compare against the original result even after database state changes
- Source: entries/2026/05/29/write-skew-detection-ssi_database-read_predicate.md

### [ACCEPT] predicate-exceptions-silently-skipped
If the predicate function throws on a `(key, value)` pair in `read_predicate`, that pair is silently excluded from results via bare `except Exception: pass` — no logging, no error propagation
- Source: entries/2026/05/29/write-skew-detection-ssi_database-read_predicate.md

### [ACCEPT] empty-predicate-match-still-locks
A predicate that matches zero keys still appends a predicate lock to `tx._predicate_locks`, enabling phantom detection if a concurrent transaction later inserts a key that would match
- Source: entries/2026/05/29/write-skew-detection-ssi_database-read_predicate.md

### [ACCEPT] read-predicate-deps-only-for-committed
`read_predicate` records dependency graph edges only for keys in the committed snapshot (`k in snap`), not for keys the transaction itself created via `write()` — avoiding self-dependencies
- Source: entries/2026/05/29/write-skew-detection-ssi_database-read_predicate.md

### [ACCEPT] ssi-read-only-skip-validation
Read-only transactions (empty write set) bypass all conflict detection in `commit()` and always commit successfully — they cannot cause write skew because they don't write
- Source: entries/2026/05/29/write-skew-detection-ssi_database.md

### [ACCEPT] ssi-phantom-detection-uses-predicate-reevaluation
Phantom detection re-evaluates each stored predicate function against current committed state at commit time, comparing both key sets and values to the snapshot-time result
- Source: entries/2026/05/29/write-skew-detection-ssi_database.md

### [ACCEPT] ssi-writes-deletes-mutually-exclusive
`write()` discards any pending `delete()` for the same key and vice versa; a key is in `tx._writes` XOR `tx._deletes`, never both simultaneously
- Source: entries/2026/05/29/write-skew-detection-ssi_database.md

### [ACCEPT] ssi-commit-returns-dict-not-exception
`SSIDatabase.commit()` returns a structured dict `{"committed": bool, "reason": str|None, "conflicts": list}` rather than raising, letting callers inspect what conflicted for retry logic
- Source: entries/2026/05/29/write-skew-detection-ssi_database.md

### [ACCEPT] ssi-pessimistic-mode-aborts-at-read
When `_pessimistic` is true, `_check_read_conflict` raises `RuntimeError` and sets `tx._status = "aborted"` at read time rather than waiting for commit — eager detection vs optimistic validation
- Source: entries/2026/05/29/write-skew-detection-ssi_database.md


