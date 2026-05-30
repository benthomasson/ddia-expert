# Topic: The SSTable format in `lsm.py` stores its sparse index as a footer; explore what happens if the footer's 12-byte trailer is corrupted vs. a mid-file data entry

**Date:** 2026-05-29
**Time:** 10:06

## SSTable Footer Corruption vs. Mid-File Data Corruption

### The file layout

The `SSTable.write()` method (lines 83–103 of `lsm.py`) produces a file with three regions:

```
[ data entries... | sparse index entries... | footer_start (8B) | count (4B) ]
                  ^                          ^--- trailer (12 bytes) ---------^
                  |
                  footer_start points here
```

The last 12 bytes — the **trailer** — are the anchor for the entire file. Every read operation begins by seeking to those 12 bytes to find the sparse index, which in turn locates data entries.

---

### Scenario 1: Corrupted trailer (last 12 bytes)

`load_index()` at line 106 does:

```python
f.seek(-12, 2)  # last 12 bytes: footer_start(8) + count(4)
footer_start = struct.unpack(">Q", f.read(8))[0]
count = struct.unpack(">I", f.read(4))[0]
```

There are no checksums, no magic bytes, no validation. A corrupted trailer creates several failure modes depending on *what* the garbage bytes decode to:

| Corrupted field | Decoded as | Result |
|---|---|---|
| `footer_start` → huge value | Offset past EOF | `f.seek()` succeeds (Python allows it), then `f.read(4)` returns `b""`, `struct.unpack` raises `struct.error` — **hard crash** |
| `footer_start` → points into data region | Valid-looking offset into entry data | The loop at lines 112–117 interprets raw key/value bytes as index entries — garbage offsets in `_sparse_index`. Subsequent `get()` calls seek to random file positions. Returns **wrong data silently** or crashes on malformed reads |
| `count` → 0 | Zero index entries | `_sparse_index` stays empty. `get()` at line 119 hits the `if not self._sparse_index: return False, None` guard — **every key appears missing**. Silent total data loss |
| `count` → very large | e.g., 4 billion | The loop tries to read billions of index entries, consuming memory until it hits a short read and crashes on `struct.unpack` — **hard crash or OOM** |

The critical insight: **a corrupted trailer makes the entire SSTable unrecoverable.** There's no fallback — no redundant pointer, no magic number to scan for, no way to discover where the sparse index starts without those 12 bytes.

---

### Scenario 2: Corrupted mid-file data entry

`_read_entry()` at lines 181–196 reads each entry as `[4B klen][key][4B vlen][value]`. Corruption here has a narrower blast radius but a subtle secondary effect:

**If the key-length field is corrupted:**
- A too-large `klen` → `f.read(klen)` returns fewer bytes than requested → the short-read guard (`if len(k) < klen: return None`) fires → `_scan_range_for_key` at line 140 breaks out of the loop. Entries *after* the corruption in that scan range are **silently skipped**, but the SSTable doesn't crash.
- A too-small `klen` → the reader consumes part of the key as the value-length header → **stream misalignment**. Every subsequent entry in the scan is misinterpreted. Depending on what bytes land in `decode("utf-8")`, you get either a `UnicodeDecodeError` (hard crash) or silently wrong keys/values.

**If the value bytes are corrupted but lengths are intact:**
- The reader has no way to detect this. The wrong value is returned as-is — **silent data corruption**. The stream stays aligned, so subsequent entries are fine.

**The sparse index limits the damage.** Because `get()` at lines 119–130 only scans between two adjacent sparse index offsets (`start_off` to `end_off`), a corrupted entry only affects lookups for keys in that one segment. Keys indexed by other sparse index entries are found via clean segments — they're unaffected.

---

### The asymmetry

| | Corrupted trailer | Corrupted data entry |
|---|---|---|
| **Blast radius** | Entire SSTable | One sparse-index segment (up to `sparse_index_interval` entries, default 16) |
| **Detection** | Usually crashes (`struct.error`) | Often silent (wrong data, skipped entries) |
| **Recovery possible?** | No — the anchor is gone | Partial — other segments still readable |
| **Defense in real systems** | Magic numbers, checksums on footer | Per-block CRCs (e.g., LevelDB uses CRC32 on every block) |

The code has zero integrity checking — no checksums, no magic bytes, no redundant metadata. This is typical of a teaching implementation, but it makes the trailer corruption especially catastrophic since that single 12-byte anchor is an unprotected single point of failure for the entire file.

---

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:compact` — Compaction rewrites SSTables; trace how a corrupted SSTable propagates (or doesn't) through the merge
- [general] `leveldb-block-format-checksums` — LevelDB adds a CRC32 and type byte to every block; compare how it survives the same corruption scenarios
- [function] `log-structured-merge-tree/lsm.py:scan_all` — Uses `_footer_start()` as the stop boundary; if the trailer is corrupt, `scan_all` reads index bytes as data entries
- [general] `sstable-bloom-filter-integration` — How a Bloom filter layer would short-circuit lookups before hitting a corrupted data segment
- [file] `log-structured-merge-tree/lsm.py` — The WAL replay logic (lines 30–50) has similar length-prefixed framing with the same corruption vulnerabilities; compare the two parsers

## Beliefs

- `sstable-trailer-single-point-of-failure` — The SSTable's 12-byte trailer (`footer_start` + `count`) is an unprotected single point of failure; its corruption makes the entire file's data and index unreachable
- `sstable-no-integrity-checks` — The SSTable reader performs no checksums, magic-byte validation, or bounds-checking on decoded offsets; all corruption is either silent or surfaces as unhandled exceptions
- `sparse-index-limits-corruption-blast-radius` — A corrupted data entry only affects lookups within one sparse index segment (default 16 entries); the sparse index isolates damage to the segment between two adjacent index offsets
- `sstable-short-read-guards-prevent-overrun` — `_read_entry()` checks for short reads on every field (`len(hdr) < 4`, `len(k) < klen`, etc.), which prevents buffer overruns but causes silent skipping of all subsequent entries in the scan range

