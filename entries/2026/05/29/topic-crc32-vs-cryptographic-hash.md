# Topic: CRC32 detects accidental corruption but not adversarial tampering; consider when a stronger integrity check (SHA-256) would be warranted

**Date:** 2026-05-29
**Time:** 06:27

# CRC32 vs. SHA-256: Integrity Checks in the Codebase

## The Two Regimes

This codebase uses two distinct integrity strategies, and the split is deliberate — it mirrors the threat models of the systems being implemented.

### CRC32: Guarding Against Disk and Network Accidents

Three storage engines use `zlib.crc32` to detect accidental corruption — bit flips, partial writes, torn pages:

- **WAL** (`write-ahead-log/wal.py:31`) — CRC covers `op_type + key + value`, stored alongside each record. On read (`wal.py:54`), the CRC is recomputed and compared; a mismatch raises `ValueError` at line 56.
- **Bitcask** (`log-structured-hash-table/bitcask.py:142`) — CRC32 is the first field in every record header (`bitcask.py:10`: `HEADER_FMT = "!III"  # crc32, key_size, value_size`). Reads validate at `bitcask.py:99` and `bitcask.py:182`.
- **B-tree** (`b-tree-storage-engine/btree.py:176`) — A `_checksum` method wraps `zlib.crc32`, used to protect WAL entries for page-level recovery (`btree.py:133`, verified at `btree.py:164`).

CRC32 is the right tool here. These are single-node storage engines where the threat is hardware — a flipped bit on disk, a kernel that didn't flush, a crash mid-write. CRC32 is fast (hardware-accelerated on most CPUs), adds only 4 bytes per record, and catches random corruption with high reliability.

**But CRC32 is not cryptographic.** It's a linear function — given a message and its CRC, an attacker can compute the CRC of a modified message without knowing the original. This means:

- A malicious actor with write access to the data file can alter a record *and* fix up its CRC to match.
- There's no authentication — CRC proves nothing about who wrote the data.
- Collision resistance is weak: at 32 bits, birthday collisions are trivial to find deliberately.

For a single-process storage engine running on a trusted local disk, none of this matters. The adversary model is "cosmic rays and power failures," not "someone editing my data files."

### SHA-256: When You Don't Trust the Other Side

The Merkle tree (`merkle-tree/merkle_tree.py:11-12`) uses SHA-256 for a fundamentally different reason:

```python
def _sha256(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()
```

Merkle trees exist for **anti-entropy** — comparing data across replicas that may be on different machines, controlled by different processes, separated by a network. The threat model shifts from accidental corruption to:

1. **Untrusted replicas** — In a distributed system, you can't assume the other node is honest. A replica could have been compromised, or could be running buggy software that silently corrupts data.
2. **Tamper-evident proofs** — `verify_proof` at `merkle_tree.py:169` lets a client verify that a specific data block is part of a tree *without downloading the entire dataset*. This only works if the hash function is collision-resistant — if an attacker could find two blocks with the same hash, the proof would be meaningless.
3. **Cross-network verification** — When hashes travel over a network, an adversary could intercept and modify them. SHA-256's preimage resistance makes this computationally infeasible.

### The Notable Gaps

Two things stand out:

**The SSTable implementation (`sstable-and-compaction/sstable.py`) has no integrity checks at all** — no CRC, no hash. It validates the magic bytes (`MAGIC = b"SSTB"` at line 9, checked at line 117), but that only catches "this isn't an SSTable file," not "this SSTable has been corrupted." A single bit flip in a value would be silently returned. Real SSTable implementations (LevelDB, RocksDB) include per-block CRC32 checksums for exactly this reason.

**The hash-index Bitcask variant (`hash-index-storage/bitcask.py`) also lacks CRC protection**, unlike its sibling in `log-structured-hash-table/bitcask.py`. It uses a simpler `HEADER_FORMAT = "<dII"` (timestamp, key_size, value_size) with no checksum field. A corrupted record would be read without error.

### When Would You Upgrade?

You'd switch from CRC32 to a cryptographic hash when the trust boundary changes:

| Scenario | CRC32 | SHA-256 |
|----------|-------|---------|
| Local WAL on a trusted disk | Sufficient | Overkill |
| Replicated WAL across nodes | Insufficient | Needed |
| SSTable served by a trusted process | Sufficient (if present) | Overkill |
| SSTable downloaded from a remote replica | Insufficient | Needed |
| B-tree page cache on local disk | Sufficient | Overkill |
| Backup verification after network transfer | Insufficient | Needed |

The pattern: CRC32 protects against **accidents on a trusted path**. SHA-256 protects against **malice or distrust across a boundary**. The codebase already demonstrates both — it just applies them to different components based on the appropriate threat model.

## Topics to Explore

- [function] `merkle-tree/merkle_tree.py:verify_proof` — How Merkle inclusion proofs work and why collision resistance matters for their security guarantee
- [file] `sstable-and-compaction/sstable.py` — Notable absence of any integrity checksums; compare with LevelDB's block-level CRC design
- [file] `hash-index-storage/bitcask.py` — Compare this CRC-less Bitcask variant with the checksummed version in `log-structured-hash-table/bitcask.py`
- [general] `byzantine-fault-tolerance` — How PBFT uses `compute_digest` to handle integrity in a fully adversarial environment where nodes may lie
- [function] `write-ahead-log/wal.py:_encode_record` — Study what exactly the CRC covers (op_type + key + value but not seq_num) and why that choice matters for corruption detection

## Beliefs

- `wal-crc-covers-payload-not-header` — The WAL CRC at `wal.py:31` checksums `op_type + key + value` but not the sequence number or record length, meaning header corruption (e.g., wrong seq_num) would go undetected
- `sstable-has-no-integrity-check` — `sstable.py` validates the magic bytes but has no per-entry or per-block checksum, so data corruption in SSTable files would be silently returned to the caller
- `merkle-tree-uses-sha256-for-cross-replica-verification` — The Merkle tree uses SHA-256 specifically because it supports tamper-evident proofs (`verify_proof`) across trust boundaries, a property CRC32 cannot provide
- `hash-index-bitcask-lacks-crc` — `hash-index-storage/bitcask.py` has no CRC field in its header format (`<dII`), unlike `log-structured-hash-table/bitcask.py` which includes CRC32 as the first header field (`!III`)

