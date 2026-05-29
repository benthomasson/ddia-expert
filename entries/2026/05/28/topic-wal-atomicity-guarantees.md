# Topic: Whether a single `write()` call actually provides atomicity on common filesystems (ext4, APFS) and what happens with partial writes across page boundaries

**Date:** 2026-05-28
**Time:** 19:23

Now I have comprehensive material. Let me compose the explanation.

# Does a Single `write()` Call Provide Atomicity? What Happens Across Page Boundaries?

## The Short Answer

**No — a single `write()` call does not provide atomicity on any common filesystem.** POSIX guarantees that `write()` on a regular file is *atomic with respect to other reads and writes on the same file descriptor* (i.e., no interleaving from concurrent threads), but it makes **no guarantee** that the bytes will land on disk as an all-or-nothing unit. The kernel can flush partial data to disk at any time, and a crash can leave you with a prefix of what you wrote.

This matters deeply because the DDIA reference implementations rely on single-`write()` atomicity for batch commit safety — and the reliance is explicitly called out as a known limitation.

## Where the Codebase Relies on `write()` Atomicity

The critical site is `append_batch` in `write-ahead-log/wal.py`. The function buffers all operations plus a trailing `COMMIT` record into a single `bytearray`, then writes it in one call:

```python
buf = bytearray()
for op_type, key, value in operations:
    self._seq_num += 1
    buf.extend(_encode_record(...))
self._seq_num += 1
commit_seq = self._seq_num
buf.extend(_encode_record(commit_seq, OP_COMMIT, b"", b""))
self._fd.write(bytes(buf))          # single write() call
self._do_sync(force=True)           # always force fsync for batches
```

The entry for `append_batch` (line 46) is honest about this: *"This doesn't guarantee atomicity at the filesystem level, but it minimizes the window for partial writes."* The `topic-wal-commit-semantics` entry (line 37–38) is even more direct: *"for large batches, the OS may flush partial data to disk before crashing."*

## What Actually Happens at the Filesystem Level

### The Kernel Path

When you call `write(fd, buf, N)`:

1. The data is copied into the kernel's **page cache** — 4 KB pages on both Linux and macOS.
2. `write()` returns successfully once the copy is complete. The data is **not on disk yet**.
3. The kernel's writeback daemon (`pdflush`/`kworker` on Linux, a similar mechanism on macOS) eventually flushes dirty pages to the disk controller.
4. `fsync()` forces all dirty pages for this file to stable storage before returning.

The critical detail: the kernel writes dirty pages to disk **independently**. If your `write()` call spans multiple 4 KB pages, those pages can reach the disk at different times. A power failure between page flushes leaves you with a **partial write**.

### ext4 (Linux)

ext4 in its default `data=ordered` mode writes file data before committing metadata to the journal. But within a single file's data, **there is no ordering guarantee across pages**. A 12 KB write spanning pages P1, P2, P3 could persist as:
- P1 and P3 flushed, P2 still in page cache → crash leaves a "hole" of stale/zero data in the middle
- P1 flushed, P2 and P3 not → classic prefix-only partial write
- All three flushed but in reverse order → P3 on disk, P1 and P2 not

The `topic-directory-fsync-semantics` entry (lines 34–37) confirms this: ext4's journal *"protects filesystem consistency (no corruption), but not durability of recent operations."*

### APFS (macOS)

APFS uses copy-on-write, which changes the failure modes but doesn't eliminate them. The `topic-directory-fsync-semantics` entry (lines 44–46) notes that APFS provides stronger implicit guarantees for *metadata* operations (renames are durable without directory fsync), but for data writes, the same fundamental issue applies: `write()` goes through the unified buffer cache, and `fsync()` / `F_FULLFSYNC` is required to force data to stable storage.

APFS does batch metadata+data into atomic "container superblock" updates during checkpoints, which can make small writes *appear* atomic in practice — but this is not a documented guarantee, and relying on it would be a correctness bug.

### The Sector-Level Wrinkle

Modern disks write in 512-byte or 4096-byte sectors. A write that falls within a single sector is typically atomic at the hardware level (the disk controller commits the sector or doesn't). But a write spanning multiple sectors can result in a **torn write** — some sectors committed, others not.

For the WAL's record format (minimum 25 bytes per record, as documented in the `_encode_record` entry), a single small record fits in one sector and is likely hardware-atomic. But a batch of records will almost certainly span sector boundaries.

## How the Codebase Defends Against Partial Writes

The WAL has a two-layer defense, implemented in `_read_record` (`wal.py:37–56`):

**Layer 1 — Length-prefixed framing detects truncation.** Each record starts with a 4-byte length. If the reader gets fewer bytes than declared, it returns `None` (treated as EOF). From the `_read_record` entry (lines 44–47):

```python
record_data = f.read(record_length)
if len(record_data) < record_length:
    return None          # torn write → treat as EOF
```

**Layer 2 — CRC32 detects corruption within complete-looking records.** If all bytes are present but garbled (the length was written but the payload is garbage), the CRC check catches it and raises `ValueError`. The `topic-wal-checksum-format` entry (lines 30–32) explains the distinction: *"Corruption to key/value data → CRC mismatch → ValueError... Corruption to the length prefix → short/wrong read → returns None."*

This means a partial write from a crashed batch manifests as either:
- A truncated record (caught by length check → `None`)
- A complete-looking record with wrong content (caught by CRC → `ValueError`)

Either way, recovery stops at the corruption boundary.

## The Atomicity Gap for Batches

Here's where it gets dangerous. The `topic-wal-commit-semantics` entry (lines 37–41) traces the exact failure mode:

1. `append_batch` writes all ops + COMMIT in a single `write()` call
2. A crash interrupts the write after some ops but before the COMMIT record reaches disk
3. On recovery, those orphaned ops are on disk without a COMMIT

**The `replay()` method does not check for COMMIT markers.** It returns all `PUT`/`DELETE` records that pass CRC validation, regardless of whether they belong to a committed batch (documented in the WAL entry, line 115). This means partial batches from a crash are replayed as if they were valid — **violating batch atomicity**.

The `iterate()` method *does* preserve COMMIT markers, so a caller who needs real atomicity can build correct replay logic on top of it. But the default `replay()` API has this gap.

## The `fsync()` Doesn't Help with Atomicity

A common misconception: `fsync()` makes data durable, but it doesn't make the preceding `write()` atomic. If a crash occurs *during* the `write()` (before `fsync()` is reached), you get a partial write. If a crash occurs *during* `fsync()`, you might get any subset of the dirty pages committed to disk.

The `append_batch` entry (line 67) identifies another subtle issue: *"if `write()` succeeds but `fsync()` fails, the sequence numbers have already been incremented and the partial data may be in the OS page cache. There's no rollback."*

## What a Production Implementation Would Do

Production WAL implementations (LevelDB, PostgreSQL, etcd) use **block-aligned records** and **CRC-protected framing** so that partial writes always land on a detectable boundary. The key techniques:

1. **COMMIT-aware replay**: Only apply records that are followed by a valid COMMIT marker (the `topic-wal-crash-recovery-correctness` entry describes this algorithm)
2. **Block-aligned writes**: Pad records to sector/page boundaries so torn writes always fall between records, never within one
3. **Write-to-temp + fsync + atomic rename**: For any non-append operation (truncation, compaction), never modify files in place

The B-tree engine's `WAL.commit` (`btree.py`) gets the ordering right: it fsyncs the data file *before* truncating the WAL (entry line 30–31), so a crash mid-truncation just replays idempotent page writes. The standalone WAL's `truncate()` does not have this safety — it rewrites files in place.

---

## Topics to Explore

- [function] `write-ahead-log/wal.py:append_batch` — Trace the exact byte counts for a realistic batch to determine whether typical batches fit within a single filesystem page (4 KB) and are therefore *practically* atomic on most hardware
- [general] `block-aligned-wal-records` — Compare this record format with LevelDB's 32 KB block-aligned log format, which ensures torn writes always land at block boundaries and makes recovery O(block-size) instead of scanning from file start
- [function] `write-ahead-log/wal.py:replay` — The actual implementation (lines 200–264, partially unobserved) to confirm whether replay does any COMMIT-aware filtering or truly ignores batch boundaries
- [general] `o-direct-and-dio` — How `O_DIRECT` bypasses the page cache entirely and gives the application direct control over write alignment and sector-boundary atomicity
- [file] `b-tree-storage-engine/btree.py` — The B-tree WAL uses page-sized writes (`write_page`) which are naturally aligned to filesystem pages, giving it stronger practical atomicity than the variable-length standalone WAL

## Beliefs

- `single-write-not-atomic-across-pages` — A single `write()` call that spans multiple filesystem pages (4 KB) provides no atomicity guarantee on ext4 or APFS; the kernel can flush pages independently, and a crash can leave a subset of pages persisted
- `wal-batch-relies-on-practical-atomicity` — `append_batch()` uses a single `write()` call to minimize the partial-write window, but the code and its documentation acknowledge this is not a true atomicity guarantee, only a best-effort optimization
- `replay-does-not-enforce-commit-boundaries` — `replay()` returns all CRC-valid PUT/DELETE records regardless of whether a COMMIT marker follows them, so a crash mid-batch produces orphaned records that are replayed as if committed
- `fsync-provides-durability-not-atomicity` — `os.fsync()` guarantees that previously written data reaches stable storage, but does not make the preceding `write()` atomic; a crash during write or fsync can leave partial data on disk
- `sector-aligned-writes-are-hardware-atomic` — Writes that fall entirely within a single disk sector (512 B or 4 KB) are typically atomic at the hardware level, so small individual WAL records (minimum 25 bytes) are practically safe from torn writes

