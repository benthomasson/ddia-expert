# Topic: DDIA Chapter 3 discusses alternative crash-safety strategies; compare this implementation against the textbook models

**Date:** 2026-05-29
**Time:** 10:34

# DDIA Chapter 3 Crash-Safety Strategies: Textbook vs. This Implementation

DDIA Chapter 3 presents three foundational crash-safety strategies for storage engines: **write-ahead logging (WAL)**, **append-only / copy-on-write structures**, and **checksummed records with recovery**. This codebase implements all three — but with varying fidelity to the textbook models. The gaps are instructive: they highlight exactly which details separate a correct description from a correct implementation.

---

## 1. Write-Ahead Logging (WAL)

### What DDIA says

The textbook rule is simple: **before any data structure is modified on disk, the intended change must be written to a durable, sequential log.** If the system crashes mid-mutation, the log is replayed to restore consistency. The key word is *durable* — the WAL record must survive the same crash that interrupted the data write.

### What the code does

The codebase has **three separate WAL implementations**, each with a different durability posture:

| WAL | Sync strategy | Crash-safe? |
|-----|--------------|-------------|
| Standalone (`write-ahead-log/wal.py`) | `flush()` + `os.fsync()` via `_do_sync` (line ~127) | **Yes** (in sync/batch modes) |
| B-tree embedded (`b-tree-storage-engine/btree.py`) | `flush()` + `os.fsync()` per entry (line ~137) | **Yes** |
| LSM tree (`log-structured-merge-tree/lsm.py`) | `flush()` only (line ~26) | **No** |

The **standalone WAL** is the most textbook-correct. Its `_do_sync` method at `wal.py:127-134` implements three configurable sync modes — `"sync"` (fsync every write), `"batch"` (fsync every N writes), and `"none"` (never fsync). The `sync` mode matches the textbook model exactly. The `batch` mode corresponds to what databases call **group commit** — a deliberate tradeoff where a small window of writes can be lost in exchange for throughput. DDIA mentions this tradeoff in the context of PostgreSQL's `synchronous_commit` setting.

The **B-tree WAL** (`btree.py`) implements a **page-level physical WAL** — each entry is a full page image (after-image), not a logical operation. The commit protocol follows the textbook perfectly:

1. Log all page images to WAL, fsync each (`log_write`, line ~137)
2. Write pages to data file, flush only (`write_page`, line ~80)
3. Fsync the data file, *then* truncate WAL (`commit`, line ~139)

This ordering is critical: the data file is durable before the WAL is cleared, so recovery can always replay if needed. The `recover` method (line ~150) replays valid entries idempotently — exactly the redo-recovery model DDIA describes for B-trees.

The **LSM WAL** is the outlier. At `lsm.py:26`, `append` calls only `self._fd.flush()` — pushing data from Python's buffer to the kernel page cache, but never to disk. A power failure loses every WAL entry still in the page cache. This contradicts the fundamental WAL invariant: the log entry must be durable before the operation is acknowledged.

### Textbook gap: The "write-ahead" in WAL means *durable-ahead*

The LSM WAL illustrates a common misunderstanding. Writing to a log file before modifying the data structure is necessary but not sufficient — the write must be **fsync'd** to stable storage. `flush()` without `fsync()` is the software equivalent of writing on a whiteboard and assuming it's permanent.

---

## 2. Redo-Only vs. Undo vs. ARIES

### What DDIA says

Chapter 3 describes WAL-based recovery but doesn't dive deep into the redo/undo distinction. The textbook treatment of ARIES (in database internals literature) identifies three approaches:

- **Redo-only**: Log the new values (after-images). Replay forward on recovery.
- **Undo-only**: Log the old values (before-images). Roll back uncommitted changes.
- **ARIES (redo + undo)**: Log both. First redo everything to crash-time state, then undo uncommitted transactions.

### What the code does

Every WAL in this codebase is **redo-only**. The `WALRecord` dataclass (`wal.py:22`) stores `(seq_num, op_type, key, value)` where `value` is the *new* value — there's no field for what was there before. The B-tree WAL stores full page after-images. Neither can reverse an operation.

This is the right choice for these engines. As the entry at `topic-redo-vs-undo-logging.md` explains: these are either **append-only** structures (Bitcask, LSM SSTables) where there's nothing to undo in-place, or **WAL-protected page stores** (B-tree) where uncommitted work is simply discarded by not replaying it. The MVCC database (`snapshot-isolation/mvcc_database.py`) handles transaction isolation via version chains at a higher level, not via WAL-level undo.

The tradeoff: redo-only recovery time is proportional to the number of records since the last checkpoint. The standalone WAL addresses this with `checkpoint` records (line ~160) and `truncate` (line ~170). The B-tree WAL truncates on every commit. The LSM WAL truncates after each flush to SSTable — but as noted above, the truncation-before-SSTable-durable race makes this dangerous.

---

## 3. Append-Only and Log-Structured Designs

### What DDIA says

DDIA presents append-only structures as inherently crash-safe: since data is never overwritten, a crash at worst leaves a partial record at the tail of the file. The log *is* the data structure, and recovery means scanning to find the last complete record.

### What the code does

**Bitcask** (`hash-index-storage/bitcask.py`) is the purest implementation of this model. The data file is opened in append mode (`"ab"`), and with `sync_writes=True` (the default), every write is fsync'd (line ~95-96). Recovery (`_scan_data_file`) scans the file sequentially and rebuilds the in-memory hash index. This matches the textbook perfectly.

**LSM SSTables** are immutable once written — also matching the textbook. But the *creation* of an SSTable is a crash-safety gap. `SSTableWriter.finish()` in `sstable.py` calls `close()` without any preceding `fsync()`. The SSTable data may still be in the page cache when `close()` returns. If the WAL is truncated after the flush (as `lsm.py:_flush` does), a crash can lose both the WAL entries and the SSTable — the data exists nowhere.

### Textbook gap: Append-only safety requires *complete* writes

The textbook claim "append-only is crash-safe" assumes the append reached stable storage. Without fsync, an append-only file has the same vulnerability as any other file — data in the page cache is volatile.

---

## 4. Checksums and Corruption Detection

### What DDIA says

Chapter 3 discusses checksums in the context of detecting torn writes (partial page writes due to crashes). LevelDB and RocksDB use per-block CRC32 checksums. B-trees may use page-level checksums.

### What the code does

The implementations split into two tiers:

**With checksums:**
- Standalone WAL (`wal.py`): CRC32 per record (line ~22). Recovery rejects entries where the checksum doesn't match, stopping at the first corruption.
- B-tree WAL (`btree.py`): CRC32 per page image (line ~170 in `recover`). Same break-on-corruption strategy.

**Without checksums:**
- LSM WAL (`lsm.py`): No checksums at all. Recovery relies on length-prefix framing and stops at the first short read.
- SSTable files: No per-entry or per-block checksums. The 12-byte footer (footer_start + count) is an unprotected single point of failure — if corrupted, the entire SSTable is unrecoverable (`topic-sstable-footer-integrity.md`).
- Bitcask data files: No checksums. Bitflips go undetected.

The **break-on-corruption** strategy (stop replaying at first bad record) is reasonable for tail corruption from torn writes, but dangerous for mid-file corruption: valid records after the corruption point are silently lost.

---

## 5. The Directory Fsync Gap

### What production systems do

LevelDB, RocksDB, and SQLite all fsync the parent directory after creating new files. This ensures the *directory entry* — not just the file contents — is durable. Without this, a newly created file can vanish on crash even if its contents were fsync'd.

### What the code does

**No module in this codebase fsyncs a directory file descriptor.** The entry at `topic-directory-fsync-gap.md` documents this comprehensively. The pattern `os.open(dir, os.O_RDONLY)` + `os.fsync(dir_fd)` appears nowhere. This affects:

- WAL rotation (`wal.py:_rotate`): New WAL segment can vanish
- SSTable creation: New SSTable can vanish
- After compaction: Output SSTable can vanish

The most dangerous compound failure: SSTable created (without directory fsync) → WAL truncated → crash → both gone.

---

## 6. Platform-Specific Durability (macOS)

The entire codebase uses `os.fsync()`, which on macOS/APFS may **not flush the disk write cache**. True durability on macOS requires `fcntl(fd, F_FULLFSYNC)`, which is never used. On this platform (Darwin 24.4.0), every `fsync()` call in the codebase provides weaker guarantees than the code implies. Additionally, `os.fdatasync()` is unavailable on macOS, so the optimization discussed in `topic-fdatasync-vs-fsync-optimization.md` would need a platform fallback.

---

## Summary: Implementation vs. Textbook

| Strategy | DDIA textbook | This codebase | Gap |
|----------|--------------|---------------|-----|
| WAL durability | fsync before ack | 2 of 3 WALs fsync; LSM WAL does not | LSM WAL is not crash-safe |
| WAL → data ordering | WAL durable before data written | B-tree does this correctly; LSM does not | LSM can lose both WAL and SSTable |
| Redo-only logging | Appropriate for append-only engines | Used consistently | Correct choice |
| Checksums | Per-block CRC | WALs have CRC; SSTables and Bitcask do not | Silent corruption possible in SSTables |
| Directory fsync | Required after file creation | Never done | New files can vanish on crash |
| Group commit | Documented tradeoff | Standalone WAL batch mode | Correctly implemented |
| Append-only safety | Crash leaves partial tail record | Bitcask correct; SSTables not fsync'd | SSTable creation has durability gap |

The **B-tree storage engine** and **standalone WAL** are the closest to textbook-correct. The **LSM tree** has the most significant gaps — its WAL provides no crash durability, and the flush-to-SSTable path can lose data in both the WAL and the SSTable simultaneously. The **hash-index Bitcask** is correct when `sync_writes=True` but has no checksums.

---

## Topics to Explore

- [function] `b-tree-storage-engine/btree.py:commit` — The textbook-correct WAL commit fence: fsync data file, then truncate WAL. Compare with the LSM tree's `_flush` to see the ordering violation.
- [function] `log-structured-merge-tree/lsm.py:_flush` — The most dangerous code path: WAL truncation before SSTable is durable. Trace exactly when data can exist in neither location.
- [general] `aries-vs-redo-only-recovery` — This codebase uses redo-only everywhere; explore what ARIES-style redo+undo would add (concurrent transaction abort, fine-grained page LSNs) and why it's overkill here.
- [file] `sstable-and-compaction/sstable.py` — No fsync, no checksums, unprotected footer — compare with LevelDB's table format (per-block CRC, magic number, metaindex block) to see the production hardening this teaching implementation omits.
- [general] `write-to-temp-then-rename-pattern` — Production databases use atomic rename (`write-temp → fsync → rename → fsync-dir`) for safe file creation. None of these implementations use this pattern.

## Beliefs

- `btree-wal-is-textbook-correct` — The B-tree WAL follows the DDIA textbook protocol: fsync WAL entry before data write, fsync data file before WAL truncation, with CRC32 checksums for torn-write detection
- `lsm-wal-violates-write-ahead-invariant` — The LSM WAL calls `flush()` without `fsync()` at `lsm.py:26`, violating the fundamental WAL invariant that log entries must be durable before the operation is acknowledged
- `lsm-flush-has-dual-loss-window` — The LSM `_flush` path can lose data in both the WAL (truncated) and SSTable (not fsync'd) simultaneously, because neither is made durable before the other is destroyed
- `no-directory-fsync-in-codebase` — No implementation fsyncs parent directories after file creation, leaving all file-creating operations (WAL rotation, SSTable flush, compaction output) vulnerable to the directory-entry-loss failure mode
- `redo-only-is-correct-for-append-only-engines` — All three WAL implementations use redo-only logging (after-images only), which is the correct choice for append-only and log-structured engines where in-place overwrites don't occur

