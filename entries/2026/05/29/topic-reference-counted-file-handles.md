# Topic: The Erlang Bitcask implementation uses reference counting so old segment files stay open until the last reader finishes; compare with the Python implementations' approach of eagerly closing handles during compaction

**Date:** 2026-05-29
**Time:** 13:09

I can't read the target repo directly, so I'll work from the observation data. The compaction methods are cut off at line 200 in both files — that's a significant gap. Here's what the visible code reveals, plus where the observations fall short.

---

# File Handle Lifecycle: Python Bitcask vs. Erlang Reference Counting

## The Erlang Bitcask Model (for context)

In the real Erlang Bitcask, when compaction merges old segment files into a new one, the old files aren't deleted immediately. Instead, each open reader (a `bitcask:fold` iterator, for instance) holds a reference to the segment's file descriptor. The old segment file stays on disk until the last reader's reference count drops to zero. This is classic **copy-on-write with deferred cleanup** — compaction produces new files, swaps the keydir atomically, and the old files die only when nobody is looking at them anymore.

## What the Python implementations actually do

Neither Python implementation has any mechanism for this. The contrast is stark across three dimensions:

### 1. Handle caching without invalidation safety

**hash-index-storage** (`bitcask.py:38–39, 76–78`):
```python
self.file_handles: dict[int, object] = {}

def _get_reader(self, file_id: int):
    if file_id not in self.file_handles:
        self.file_handles[file_id] = open(self._data_path(file_id), "rb")
    return self.file_handles[file_id]
```

**log-structured-hash-table** (`bitcask.py:47, 138–140`):
```python
self._file_handles: dict[str, object] = {}

def _get_read_handle(self, path: str):
    if path not in self._file_handles:
        self._file_handles[path] = open(path, "rb")
    return self._file_handles[path]
```

Both maintain a flat dictionary of lazily-opened read handles. There's no reference count, no "who is currently using this handle" tracking, and no mechanism to defer closing a handle that a concurrent reader might be mid-`seek`/`read` on.

### 2. The log-structured variant partially works around this — but only for `get()`

The log-structured implementation's `get()` method (line ~168) sidesteps the shared handle entirely:

```python
def get(self, key: str) -> Optional[bytes]:
    ...
    with open(seg_path, "rb") as f:   # fresh handle every time
        f.seek(offset)
        ...
```

This opens a *new* file handle per read, which avoids position conflicts between concurrent readers. But it means the cached `_file_handles` dict is only used during startup recovery (`_scan_segment`), not for serving reads. The hash-index variant uses `_get_reader()` for all reads (line 96), so two reads to the same segment would share a file position — a bug in any concurrent scenario.

### 3. Compaction: eager close, no deferred cleanup

The `compact()` methods are cut off at line 200 in both files, so I can't show the exact deletion logic. However, the architectural signals are clear:

- **No reference counting or reader registration** exists anywhere in either file
- **No lock or mutex** protects the handle dictionaries
- The `put()` method in log-structured (line ~165) eagerly pops stale handles: `self._file_handles.pop(self._active_path, None)`
- The `compact()` call in hash-index starts by identifying `immutable_ids` (line ~200) — files that aren't the active writer — and almost certainly deletes the old data files and removes their handles from `file_handles` after merging

This means compaction will **delete segment files out from under any concurrent reader**. If a `get()` call had resolved a keydir entry to segment 3's file and offset, and compaction deletes segment 3 between the index lookup and the `read()`, the reader gets an I/O error or stale data.

### The real Erlang Bitcask handles this via three mechanisms the Python code lacks:

| Mechanism | Erlang Bitcask | Python implementations |
|-----------|---------------|----------------------|
| **Refcounted file descriptors** | Yes — each reader increments; file deleted when count hits zero | No — flat dict, no counting |
| **Atomic keydir swap** | CAS operation on ETS table | Direct dict mutation (`self.keydir[key] = ...`) |
| **Concurrent reader safety** | Readers hold refs to old generation; new readers see new generation | No generation concept; single mutable dict |

## The bottom line

These Python implementations are **single-threaded, single-process storage engines**. They model the Bitcask data format (append-only log, in-memory hash index, hint files) but intentionally omit the concurrency machinery that makes Bitcask production-grade. The eager handle management isn't a bug — it's a scope decision. Reference counting would add significant complexity (tracking reader generations, deferred file deletion, atomic keydir swaps) that's orthogonal to teaching the core append-only log + hash index concept.

**What's missing from the observations:** The compaction method bodies (lines 200+ in both files) would show exactly how old segment files are deleted and whether handles are cleaned up from the dictionaries. Without that code, I'm inferring the eager-close behavior from the absence of any deferred-cleanup infrastructure.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — The compaction body (line 200+) would reveal exactly how old segments are deleted and whether file handles are cleaned up from the `file_handles` dict
- [function] `log-structured-hash-table/bitcask.py:compact` — Compare the second implementation's compaction strategy, especially whether it writes hint files and how it updates `_index` entries pointing to merged segments
- [general] `erlang-bitcask-refcount-model` — Study the Erlang Bitcask source (`bitcask_fileops.erl`) to see how `#filestate.ref_count` manages deferred deletion and compare with the Python simplification
- [function] `log-structured-hash-table/bitcask.py:get` — Uses `with open(...)` per read instead of cached handles — an implicit concurrency-safety pattern worth contrasting with hash-index's shared-handle approach
- [general] `segment-generation-and-keydir-swap` — How production Bitcask does atomic keydir replacement during merge (ETS table swap) vs. the in-place dict mutation in both Python versions

## Beliefs

- `hash-index-bitcask-shared-read-handles` — `hash-index-storage/bitcask.py` uses a single cached file handle per segment for all reads via `_get_reader()`, making concurrent reads to the same segment unsafe due to shared seek position
- `log-structured-bitcask-fresh-handle-per-get` — `log-structured-hash-table/bitcask.py:get()` opens a fresh file handle with `with open(...)` on every read, bypassing the `_file_handles` cache entirely
- `python-bitcask-no-refcount-or-locking` — Neither Python Bitcask implementation has reference counting, reader registration, or locking — compaction can delete segment files while a concurrent reader holds a stale index entry pointing to them
- `python-bitcask-single-threaded-scope` — Both Python Bitcask implementations are designed as single-threaded teaching engines; the absence of Erlang Bitcask's concurrency machinery (refcounted FDs, atomic keydir swap) is a deliberate scope boundary, not a bug

