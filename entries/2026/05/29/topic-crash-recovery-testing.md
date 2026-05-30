# Topic: Write a test that kills the process mid-compaction (after delete, before rename) and verify whether the store can recover without data loss

**Date:** 2026-05-29
**Time:** 12:16

I don't have read access to the target repo. But the observations include grep hits and the first 200 lines, so let me work with what's available and note what's missing.

---

# Crash-During-Compaction Recovery Test

## The Scenario

The question targets a specific **danger window** in LSM tree compaction: the moment after old SSTable files have been deleted but before the new merged SSTable has been renamed into place. If the process dies in this gap, data that existed in the old SSTables could be permanently lost — unless the implementation has safeguards.

## Understanding the Compaction Flow

From the observations, we can see the `compact()` method starts at **`lsm.py:319`**. The grep results show the key sequence:

1. **Line 340**: `# Remove tombstones during compaction` — entries are merged, tombstones stripped
2. The method merges all SSTables in `self._sstables` into a single new SSTable
3. Old SSTables are deleted
4. The new SSTable replaces them

Critically, the grep for `os.rename|os.replace|shutil.move` (**no matches in `lsm.py`**) tells us something important: **the LSM module doesn't use atomic rename at all**. The two files that do use `os.rename` are in the `hash-index-storage/bitcask.py` and `log-structured-hash-table/bitcask.py` modules — not the LSM tree.

This means the compaction in `lsm.py` likely:
1. Writes the merged SSTable to a new file
2. Deletes the old SSTable files (via `os.remove` or `os.unlink`)
3. Updates `self._sstables` in memory

There's no atomic swap, so the crash window is real.

## What the Test Should Do

```python
def test_crash_mid_compaction():
    """Kill the process after old SSTables are deleted but before
    compaction completes — verify recovery doesn't lose data."""
    with tempfile.TemporaryDirectory() as d:
        # 1. Populate: create multiple SSTables with known data
        db = LSMTree(d, memtable_threshold=2, compaction_threshold=100)
        db.put("a", "1"); db.put("b", "2")   # flush → SSTable 1
        db.put("c", "3"); db.put("d", "4")   # flush → SSTable 2
        db.put("e", "5"); db.put("f", "6")   # flush → SSTable 3
        db.close()

        # 2. Record all SSTable files before compaction
        sst_files_before = [f for f in os.listdir(d) if f.endswith('.sst')]

        # 3. Monkey-patch os.remove to crash after deleting old files
        #    but before the new SSTable is registered
        original_remove = os.remove
        removed = []
        def crashing_remove(path):
            original_remove(path)
            removed.append(path)
            if len(removed) >= 2:  # After deleting 2 old SSTables
                raise SystemExit("simulated crash")

        import unittest.mock
        db2 = LSMTree(d, memtable_threshold=2, compaction_threshold=100)
        with unittest.mock.patch('os.remove', side_effect=crashing_remove):
            try:
                db2.compact()
            except SystemExit:
                pass  # "crash" happened

        # 4. Reopen and verify: can we recover ALL data?
        db3 = LSMTree(d, memtable_threshold=2, compaction_threshold=100)
        for key, expected in [("a","1"),("b","2"),("c","3"),("d","4"),("e","5"),("f","6")]:
            actual = db3.get(key)
            assert actual == expected, f"Lost {key}: expected {expected}, got {actual}"
        db3.close()
```

## Why This Matters

The existing `test_crash_recovery` at **`test_lsm.py:92`** only tests WAL replay for unflushed memtable data — it doesn't touch compaction at all. The `test_compaction` at **`test_lsm.py:68`** verifies happy-path compaction but never simulates a crash during it.

The key insight: **WAL doesn't help here**. The WAL (defined at `lsm.py:14`) protects unflushed memtable entries. Once data is flushed to SSTables, the WAL is truncated (`lsm.py`, `WAL.truncate()`). So if SSTables are deleted during compaction, that data is gone — the WAL won't have it.

## What's Missing From the Observations

I couldn't read lines 319–359 of `lsm.py`, which contain the full `compact()` implementation. To write a precise test, we need to see:

1. **Whether `compact()` deletes old files before or after writing the new one** — this determines the exact crash window
2. **Whether there's a compaction manifest or journal** — some implementations log compaction intent before starting, enabling recovery
3. **How `_sstables` is reconstructed on startup** — does `LSMTree.__init__` scan the directory for `.sst` files, or does it rely on a manifest?

Without seeing the full method, the most likely outcome is: **the current implementation will lose data on a mid-compaction crash**, because there's no compaction WAL, no manifest, and no atomic rename. This is a known limitation of simple LSM implementations and exactly the kind of thing DDIA discusses when motivating crash-safe compaction designs.

## Topics to Explore

- [function] `log-structured-merge-tree/lsm.py:compact` — The full compaction implementation; need lines 319–359 to see the delete/write/rename sequence and confirm whether there's any crash protection
- [function] `log-structured-merge-tree/lsm.py:LSMTree.__init__` — How SSTables are discovered on startup determines whether partially-completed compactions can be detected and recovered
- [general] `atomic-rename-compaction-pattern` — The standard fix: write new SSTable to a temp file, fsync, atomically rename over old, then delete inputs. Compare with the Bitcask modules (`hash-index-storage/bitcask.py:297`) that do use `os.rename`
- [file] `write-ahead-log/wal.py` — The WAL module (`wal.py:62`) has a more sophisticated design with batching, rotation, and corruption detection — could inform how to add a compaction journal to the LSM tree
- [general] `manifest-based-recovery` — How production LSM stores (LevelDB, RocksDB) use a MANIFEST file to make compaction crash-safe; the current implementation appears to lack this

## Beliefs

- `lsm-compact-no-atomic-rename` — The LSM tree's `compact()` method does not use `os.rename` or `os.replace`; no file in the LSM module appears in the grep results for atomic rename operations
- `wal-does-not-protect-flushed-data` — The WAL is truncated after each memtable flush, so data already written to SSTables has no WAL backup during compaction
- `existing-crash-test-ignores-compaction` — `test_crash_recovery` (`test_lsm.py:92`) only covers WAL replay for unflushed memtable entries, not crashes during SSTable compaction
- `compaction-removes-tombstones` — Compaction strips tombstone entries (the delete markers), as noted at `lsm.py:340`, meaning a crash after partial compaction could resurrect deleted keys if the old SSTables survive but the new one doesn't

