# Topic: Instrument both engines to count total bytes written per logical put, especially under update-heavy workloads with compaction — this quantifies the LSM-tree's main cost

**Date:** 2026-05-29
**Time:** 12:34

# Instrumenting Write Amplification: LSM-Tree vs B-Tree

Write amplification — the ratio of total physical bytes written to logical bytes intended — is the central cost of LSM-trees. Instrumenting both engines to measure it requires tracing every disk write triggered by a single `put`, including deferred writes from compaction and page splits.

## The Write Paths

### LSM-Tree (`log-structured-merge-tree/lsm.py`)

A single `put` triggers two immediate writes, plus deferred writes:

1. **WAL append** (lines 20–26): `WAL.append()` writes `4 + len(key_bytes) + 4 + len(value)` bytes — two 4-byte length headers plus the key and value payloads. This is the synchronous durability cost.

2. **Memtable flush** (deferred): When the memtable fills, `SSTable.write()` (lines 80–100) serializes every buffered entry to disk. Each entry costs `4 + len(key_bytes) + 4 + len(value)` in the data block, plus the sparse index footer entries (key + 8-byte offset every `sparse_index_interval` entries, then an 8-byte footer-start pointer and 4-byte count).

3. **Compaction** (deferred): The `LSMTree` class (starts at line 200, truncated in observations) merges SSTables. Every compaction pass **re-writes** all entries from the input SSTables into new output SSTables. This is where write amplification compounds — the same key-value pair can be rewritten once per compaction level, giving worst-case amplification of O(levels × size_ratio).

### B-Tree (`b-tree-storage-engine/btree.py`)

A single `put` writes at coarser granularity:

1. **WAL log** (lines 128–135): `WAL.log_write()` writes `12 + page_size + 4` bytes — a 12-byte header (seq + page_num + data_len), the full page image, and a 4-byte CRC32 checksum. For the default 4096-byte page, that's **4112 bytes per WAL entry**, regardless of how small the key-value pair is.

2. **Page write** (lines 78–85): `PageManager.write_page()` always writes exactly `page_size` bytes (4096 default), padded with zeroes. Updating a single byte in a leaf means rewriting the entire page.

3. **Metadata update** (lines 44–49): `_write_meta()` rewrites the full metadata page (another `page_size` bytes) whenever the tree structure changes (key count, root pointer, free list).

4. **Page splits** (deferred): When a leaf overflows, the split allocates a new page and rewrites the parent — at minimum 3 page writes (two leaves + parent), each `page_size` bytes.

## Instrumentation Strategy

### B-Tree: Already Partially Instrumented

The `PageManager` already tracks page I/O at lines 34–35:
```python
self.pages_read = 0
self.pages_written = 0
```
with a `reset_counters()` method at lines 107–108. To get total bytes written per `put`:

```python
bytes_written = pm.pages_written * pm.page_size
```

**What's missing:** WAL bytes are not counted. `WAL.log_write()` writes `12 + len(page_data) + 4` bytes per call but has no counter. Add a `bytes_written` accumulator to the `WAL` class and expose it alongside `PageManager.pages_written`.

### LSM-Tree: No Instrumentation Exists

The LSM engine has no write counters at all. You need to instrument three layers:

1. **`WAL.append()`** — accumulate `4 + len(k) + 4 + len(value)` per call
2. **`SSTable.write()`** — after writing, measure the output file size (or accumulate within the `for` loop at line 87)
3. **Compaction** — the compaction function (beyond line 200, not shown) creates new SSTables. Instrument the same way as flush: count the bytes of each output SSTable

The key metric is:

```
write_amplification = total_physical_bytes_written / sum_of_logical_put_sizes
```

where `logical_put_size = len(key) + len(value)` for each user-facing `put` call.

## Why This Matters Under Update-Heavy Workloads

When the workload is update-heavy (repeatedly overwriting the same keys):

- **LSM**: Every version of a key lives in a separate SSTable until compaction merges them. With size-tiered compaction, a key updated N times might exist in N SSTables across multiple levels. Each compaction pass rewrites surviving entries, so the same logical byte gets written once to the WAL, once during flush, and once per compaction pass that touches it. With L levels and a size ratio T, worst-case write amplification is **O(T × L)** — often 10–30× in practice.

- **B-Tree**: An update overwrites the leaf page in-place. Write amplification is bounded by **(page_size / avg_entry_size) × 2** (one WAL write + one page write). For a 100-byte entry in a 4096-byte page, that's ~80× per write, but the amplification doesn't compound with subsequent updates to the same key — there's no deferred rewriting.

The crossover: LSM wins when entries are small relative to page size (amortized flush is cheaper than a full-page rewrite), but loses under update-heavy workloads where compaction rewrites dominate. Instrumenting both engines lets you measure exactly where that crossover occurs for your data.

## Observations Gaps

The `LSMTree` class definition (starting at line 200) was truncated — the compaction logic, memtable flush trigger, and `put` method are not visible. The `sstable-and-compaction/sstable.py` file shows the SSTable format but the compaction orchestration (size-tiered and leveled merge) was also truncated at line 200. Full instrumentation requires seeing both the compaction loop and the `put` → flush → compact call chain.

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:LSMTree` — The `put`, flush, and compaction methods are beyond line 200; this is where write amplification actually compounds
- [file] `sstable-and-compaction/sstable.py` — Lines 200+ contain the compaction strategies (size-tiered vs leveled) that determine the amplification factor
- [function] `b-tree-storage-engine/btree.py:PageManager.allocate_page` — Page allocation and free-list management (lines 91–100) add hidden writes during splits
- [general] `write-amplification-benchmark` — Build a benchmark that runs N puts to the same key set on both engines, comparing `total_bytes_written / (N * avg_entry_size)` as a function of dataset size
- [function] `b-tree-storage-engine/btree.py:WAL.commit` — The WAL truncation at line 141 is a write-amplification reset point; understanding commit boundaries matters for accurate measurement

## Beliefs

- `lsm-no-write-counters` — The LSM-tree engine has no built-in byte or operation counters; all write instrumentation must be added from scratch
- `btree-has-page-counters` — The B-tree's `PageManager` already counts `pages_read` and `pages_written` (multiply by `page_size` for bytes), but WAL writes are uncounted
- `btree-page-write-always-full` — `PageManager.write_page()` always writes exactly `page_size` bytes regardless of payload size, padding with zeroes — this is the B-tree's fixed per-write cost
- `lsm-wal-entry-size` — Each LSM WAL entry is exactly `8 + len(key_utf8) + len(value)` bytes: two 4-byte big-endian length headers plus the raw payloads
- `btree-wal-entry-includes-full-page` — Each B-tree WAL entry is `16 + page_size` bytes (12-byte header + full page image + 4-byte CRC32), making WAL writes the dominant cost for small updates

