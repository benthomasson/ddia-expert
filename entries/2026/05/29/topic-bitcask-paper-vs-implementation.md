# Topic: How this implementation compares to the original Bitcask paper (missing CRC, merge triggers, expiry)

**Date:** 2026-05-29
**Time:** 12:00

# How This Implementation Compares to the Original Bitcask Paper

The repository contains **two separate Bitcask implementations** that take different stances on fidelity to the [original Bitcask paper](https://riak.com/assets/bitcask-intro.pdf). The divergences cluster around three areas: data integrity (CRC), merge/compaction triggers, and key expiry.

---

## 1. CRC / Data Integrity

**The paper** requires every record to carry a CRC32 checksum so that bit-rot and partial writes are detected on read and during merge.

| Implementation | CRC present? | Details |
|---|---|---|
| `hash-index-storage/bitcask.py` | **No** | The header format at line 10 is `<dII` — timestamp, key_size, value_size. No checksum field. A corrupted record will silently return wrong data. The `_read_record` method (line 93) trusts whatever bytes it finds. |
| `log-structured-hash-table/bitcask.py` | **Yes** | The header format at line 10 is `!III` — crc32, key_size, value_size. `_write_record` (line 142) computes `zlib.crc32(payload)` and packs it into the header. Both `_scan_segment` (line 95–100) and `get` (line 178–183) verify the CRC on read and raise `CorruptionError` on mismatch. |

The `hash-index-storage` variant is the one that departs from the paper. It relies on `os.fsync` (line 89) to ensure writes land on disk but has no mechanism to detect corruption after the fact. If a write is torn (power loss mid-write during index rebuild), `_scan_data_file` (line 113) will happily ingest garbage — the only guard is a short-read check at line 119.

---

## 2. Merge / Compaction Triggers

**The paper** describes a merge process that runs when the number of immutable (non-active) data files crosses a threshold, and it produces new merged files plus hint files for fast restart.

| Implementation | Trigger | Behavior |
|---|---|---|
| `hash-index-storage/bitcask.py` | **Manual only** | `compact()` is a public method (line 198). Nothing in `put` or `_maybe_rotate` (line 105) triggers it automatically. The caller must decide when to compact. |
| `log-structured-hash-table/bitcask.py` | **Automatic threshold** | The constructor takes `auto_compact_threshold` (default 5, line 39). Inside `put` (lines 157–160), after every write, it counts frozen segments and calls `self.compact()` when the count reaches the threshold. This is close to the paper's model. |

The `hash-index-storage` version's manual-only compaction means that in a long-running process, dead data accumulates indefinitely unless the application explicitly calls `compact()`. The paper considers automatic triggering essential because unbounded file growth degrades startup time (more files to scan) and wastes disk.

---

## 3. Key Expiry / TTL

**The paper** specifies an optional per-key expiry timestamp. During merge, expired keys are dropped just like tombstones. Neither implementation supports this:

- `hash-index-storage/bitcask.py` stores a timestamp in the header (line 10, the `d` in `<dII`), but it is only used as metadata in `KeyEntry` — nothing ever checks whether a key has expired.
- `log-structured-hash-table/bitcask.py` doesn't store a timestamp at all — its header is just CRC + sizes.

This means neither implementation can serve use cases like session stores or caches where keys should auto-evict. To add expiry, you'd need to:
1. Add a TTL field to the record header.
2. Check it in `get` (return `None` + optionally write a tombstone).
3. Skip expired keys during merge/compaction.

---

## 4. Other Notable Divergences

**Tombstone encoding.** The paper uses a dedicated tombstone marker. `hash-index-storage/bitcask.py` uses an empty-string value (`""`) as the tombstone sentinel (line 84: `self._write_record(key, "")`), which means empty strings can't be stored as legitimate values — `get` returns `None` for `""` at line 176. The `log-structured-hash-table` version uses a proper sentinel byte sequence `TOMBSTONE = b"__BITCASK_TOMBSTONE__"` (line 9), which is more robust but still technically collides if someone stores that exact byte pattern.

**Value types.** `hash-index-storage` stores string values; `log-structured-hash-table` stores raw `bytes`. The paper is byte-oriented, so the latter is closer.

---

## Summary Table

| Feature | Paper | `hash-index-storage` | `log-structured-hash-table` |
|---|---|---|---|
| CRC per record | Required | Missing | Present |
| Hint files | Required | Present | Present |
| Auto merge trigger | Described | Missing (manual only) | Present (threshold) |
| Key expiry / TTL | Optional | Missing | Missing |
| Tombstone encoding | Dedicated marker | Empty string (leaky) | Sentinel bytes |
| `fsync` on write | Recommended | Configurable | Not called |

---

## Topics to Explore

- [function] `hash-index-storage/bitcask.py:compact` — See how the merge process rewrites live entries and generates hint files, and compare to the paper's merge specification
- [function] `log-structured-hash-table/bitcask.py:compact` — Compare the auto-triggered compaction with the manual version to understand the trade-offs in merge design
- [file] `write-ahead-log/wal.py` — Uses CRC32 with a proper corruption-detection path; good reference for how the `hash-index-storage` Bitcask *should* handle integrity
- [general] `bitcask-tombstone-semantics` — The empty-string-as-tombstone design in `hash-index-storage` creates a subtle data loss bug worth investigating
- [general] `bitcask-startup-time` — Measure how hint files vs. full data-file scans affect index rebuild time as the dataset grows

## Beliefs

- `hash-index-storage-no-crc` — `hash-index-storage/bitcask.py` has no CRC or checksum in its record format; corrupted or torn records will be silently accepted during both reads and index rebuilds
- `log-structured-bitcask-has-crc` — `log-structured-hash-table/bitcask.py` includes CRC32 in every record header and validates it on both `get` reads and segment scans, raising `CorruptionError` on mismatch
- `hash-index-no-auto-compact` — `hash-index-storage/bitcask.py` never triggers compaction automatically; the caller must explicitly invoke `compact()`, so dead data accumulates unboundedly in production use
- `empty-string-tombstone-bug` — `hash-index-storage/bitcask.py` uses `""` as the tombstone sentinel, making it impossible to distinguish a deleted key from one whose value is legitimately an empty string
- `neither-bitcask-supports-expiry` — Neither Bitcask implementation supports per-key TTL or expiry, a feature described as optional in the original paper

