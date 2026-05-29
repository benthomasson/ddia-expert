# Topic: Read the original Bitcask paper (Sheehy & Smith, 2010) to compare the Erlang-specific details (keydir sharing via ETS tables, lock files, merge triggers) against these Python simplifications

**Date:** 2026-05-29
**Time:** 08:35

# Bitcask Paper vs. Python Implementations: Erlang-Specific Details

The original Bitcask paper (Sheehy & Smith, 2010) was designed for Riak, an Erlang/OTP system. Several of its core mechanisms exploit Erlang's concurrency primitives in ways that these Python implementations deliberately simplify away. Here's where the two diverge — and what's lost.

---

## 1. KeyDir Sharing via ETS Tables

**Paper**: The keydir is stored in an Erlang ETS (Erlang Term Storage) table — a shared-memory concurrent hash table built into the BEAM VM. Multiple Erlang processes (lightweight green threads) can read the keydir simultaneously without copying it, and writers perform atomic updates. When a new Erlang process opens an already-open Bitcask, it gets a reference to the *same* in-memory keydir via named ETS lookup — zero-cost sharing across the entire Riak vnode.

**Python implementations**: Both implementations use a plain `dict` scoped to a single `BitcaskStore` instance:

- `hash-index-storage/bitcask.py:35` — `self.keydir: dict[str, KeyEntry] = {}`
- `log-structured-hash-table/bitcask.py:44` — `self._index: dict[str, tuple[str, int]] = {}`

There is **no shared state** between processes or threads. The grep for `shared|concurrent|thread|process|ETS` (`grep_ets_shared_memory`) returns zero relevant hits — only incidental matches on `os.path.getsize`. Each store instance owns its keydir exclusively.

**What's lost**: In production Bitcask, the ETS-backed keydir means a Riak node can open the same Bitcask directory from dozens of vnodes or request-handling processes without duplicating the index in memory. These Python implementations would need to reconstruct the entire keydir from disk on each instantiation (which they do — see `_rebuild_index` at line 113 in `hash-index-storage/bitcask.py` and `_recover` at line 73 in `log-structured-hash-table/bitcask.py`). For a keydir with millions of entries, this startup cost is significant — exactly the problem ETS sharing solves.

---

## 2. Lock Files for Writer Exclusion

**Paper**: Bitcask enforces a single-writer invariant using OS-level lock files. Only one Erlang process can hold the write lock on a Bitcask directory at a time. Other processes can open the same directory for reads (sharing the keydir via ETS), but writes are serialized through the lock holder. The lock file also records the PID of the writer for crash detection — if the process dies, the stale lock can be broken.

**Python implementations**: Neither implementation has any locking mechanism. The grep for `lock|Lock|flock|fcntl` shows zero matches in either Bitcask file — all results are from the unrelated `fencing-tokens/` module.

- `hash-index-storage/bitcask.py` opens the active file with `open(path, "ab")` at line 70 with no lock acquisition
- `log-structured-hash-table/bitcask.py` does the same at its `_open_new_segment` method (line 120)

**What's lost**: Without locking, nothing prevents two `BitcaskStore` instances from simultaneously appending to the same data directory, corrupting each other's data files and producing an inconsistent keydir. The Python implementations are strictly single-process, single-instance tools. This is fine for educational use and matches the DDIA framing, but it means you can't safely embed them in a multi-process server.

---

## 3. Merge Triggers and Background Merge

**Paper**: Bitcask's merge (compaction) runs as a separate Erlang process, triggered automatically by configurable policies: fragmentation ratio, dead bytes threshold, file count, or time-based schedules. The merge process reads immutable (frozen) segment files, writes compacted output files with hint files, and atomically swaps keydir entries via ETS — all while the writer continues appending to the active file and readers continue serving from the old segments until the swap completes.

**Python implementations**: Both have manual compaction, but with very different trigger models:

- **`hash-index-storage/bitcask.py:194`** — `compact()` is entirely manual. The caller must invoke `store.compact()` explicitly. There is no fragmentation tracking, no dead-byte counting, no threshold-based trigger.

- **`log-structured-hash-table/bitcask.py:161-166`** — Has a basic auto-compact trigger: when the number of frozen segments exceeds `auto_compact_threshold` (default 5), compaction runs synchronously inside `put()`:
  ```python
  frozen = self._frozen_segment_paths()
  if len(frozen) >= self._auto_compact_threshold:
      self.compact()
  ```
  This is a simplified version of the paper's trigger, using only segment count — not fragmentation ratio or dead bytes.

**Critical difference**: In both Python implementations, compaction is **synchronous and blocking**. While `compact()` runs, no reads or writes can proceed. The paper's Erlang implementation runs merge as a background process with concurrent read access to old files and atomic keydir updates. The Python versions must block the caller during the entire merge operation — copying live records, writing hint files, deleting old segments, and updating the keydir all happen in the same thread of execution.

---

## 4. Other Erlang-Specific Details Simplified Away

| Paper Feature | `hash-index-storage` | `log-structured-hash-table` |
|---|---|---|
| **CRC integrity** | No checksums | CRC32 on every record (line 143) |
| **Tombstone encoding** | Empty string `""` (line 186) | Sentinel bytes `b"__BITCASK_TOMBSTONE__"` (line 7) |
| **Value types** | `str` only | `bytes` (closer to paper) |
| **Hint files** | Written during compact (line 161) | Separate `create_hint_files()` call |
| **Iterator protocol** | `keys()` returns list (line 192) | Full `__iter__` with `Iterator` support |
| **fsync policy** | Configurable `sync_writes` (line 86) | Always `flush()`, no `fsync` |

The CRC difference is notable: the paper mandates CRC checks on every read to detect bitrot and partial writes. `hash-index-storage/bitcask.py` omits this entirely, while `log-structured-hash-table/bitcask.py` implements it faithfully with `zlib.crc32` and raises `CorruptionError` on mismatch (tested at `test_bitcask.py:121-131`).

---

## Summary

These implementations capture Bitcask's core insight — an append-only log with an in-memory hash index gives you O(1) reads with high write throughput — while deliberately dropping the production machinery that makes it work in a concurrent, multi-process database server. The ETS keydir, lock files, and background merge are not incidental details; they're what make Bitcask usable as a storage engine inside Riak rather than a single-user CLI tool. Understanding what was removed is as instructive as understanding what was kept.

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — The full compaction logic (lines 194-300+) showing how keydir entries are atomically swapped and old files deleted, compared to the paper's atomic ETS update approach
- [function] `log-structured-hash-table/bitcask.py:_scan_segment` — How partial-write recovery works during crash recovery (lines 89-107), and how stopping at the first CRC mismatch mirrors the paper's append-only corruption boundary guarantee
- [general] `erlang-ets-concurrency-model` — Study how ETS tables provide concurrent read access with isolated write semantics, and why Python's GIL-protected `dict` is not equivalent (GIL doesn't help across processes)
- [file] `log-structured-hash-table/test_bitcask.py` — The test for auto-compaction (line 161) and partial-write recovery (line 178) demonstrate the two closest analogs to the paper's production behavior
- [general] `bitcask-merge-window-scheduling` — The paper describes time-window-based merge scheduling to avoid compaction during peak load — neither implementation models this, but it's critical for production use

## Beliefs

- `python-bitcask-no-writer-locking` — Neither Python Bitcask implementation enforces single-writer exclusion; concurrent instances writing to the same directory will corrupt data
- `keydir-rebuilt-on-every-open` — Both implementations rebuild the entire in-memory index from disk on startup, unlike the paper where ETS keydir persists across process opens within the same node
- `compaction-is-synchronous-and-blocking` — Both implementations run compaction in the calling thread, blocking all reads and writes until merge completes
- `only-log-structured-variant-has-crc` — `log-structured-hash-table/bitcask.py` checks CRC32 on every read and raises `CorruptionError` on mismatch; `hash-index-storage/bitcask.py` has no integrity verification
- `auto-compact-trigger-uses-segment-count-only` — `log-structured-hash-table/bitcask.py` triggers compaction when frozen segment count exceeds a threshold, ignoring the paper's fragmentation ratio and dead-byte metrics

