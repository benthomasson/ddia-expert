# File: secondary-index-partitioning/secondary_index_partitioning.py

**Date:** 2026-05-29
**Time:** 09:05

# `secondary_index_partitioning.py` — Explanation

## Purpose

This file implements the two strategies for partitioning secondary indexes described in DDIA Chapter 6: **document-partitioned** (local) indexes and **term-partitioned** (global) indexes. It provides a side-by-side comparison of the two approaches, tracking how many partitions each operation touches — the central tradeoff Kleppmann describes: scatter/gather reads vs. multi-partition writes.

The module owns the partitioning logic, index maintenance, and operation metrics. It's a teaching implementation — no persistence, no networking, no concurrency — focused on making the structural tradeoffs observable.

## Key Components

### `Document` (lines 5–9)
A thin wrapper around a primary key and a field dictionary. Notably, the DB classes don't actually use this class — they accept `pk` and `fields` separately. It exists as a domain concept but isn't wired into the API.

### `Partition` (lines 12–23)
Represents one partition shard. Holds three data structures:
- `documents`: primary key → field dict (the actual data)
- `local_index`: field name → {value → set of PKs} (used by document-partitioned strategy)
- `global_index`: field name → {value → set of PKs} (used by term-partitioned strategy)

Each partition has both index structures available, but only one is populated depending on which DB class manages it.

### `OperationResult` (lines 26–32)
Return type for all DB operations. Bundles `data` (query results or None for writes), `partitions_touched` (the key metric), and `operation` type. This is how the implementation makes the DDIA tradeoff measurable.

### `DocumentPartitionedDB` (lines 35–100)
Implements the **local index** strategy. Each partition maintains its own secondary index covering only documents stored on that partition.

- **`put`**: Hashes PK to find partition, removes old index entries if updating, stores document, adds new index entries. Touches exactly **1 partition** always.
- **`get`**: Direct PK lookup. **1 partition**.
- **`delete`**: Removes document and cleans up index entries. **1 partition**.
- **`query_by_field`**: The scatter/gather query — iterates **all partitions**, checking each local index. Always touches `num_partitions` partitions regardless of how many results exist.

### `TermPartitionedDB` (lines 103–230)
Implements the **global index** strategy. The index is itself partitioned — each index term lives on a specific partition determined by hashing or range-partitioning the term value.

- **`put`**: Stores document on PK-hashed partition, then updates index entries on potentially **different partitions** (wherever each term hashes to). Touches 1 + N partitions where N is the number of indexed field values that map to distinct partitions.
- **`get`**: Same as document-partitioned — **1 partition**.
- **`delete`**: Like `put` in reverse — removes index entries from remote partitions.
- **`query_by_field`**: Looks up the single partition that owns the queried term, retrieves PKs, then fetches documents from their home partitions. Touches **1 + K partitions** where K is the number of distinct document partitions holding results. This is the payoff — no scatter/gather.
- **`query_range`**: Scans all partitions for range matches. This degrades toward scatter/gather because range queries over hash-partitioned indexes can't be localized. Most useful with `partition_by="range"`.
- **`flush_index`**: Applies batched index updates when `async_index=True`. Simulates the real-world pattern where global index updates are asynchronous to avoid cross-partition coordination on the write path.

### `compare_strategies` (lines 233–261)
Utility function that runs an identical workload (document inserts + field queries) against both DB types and returns their stats side-by-side. This is the pedagogical payoff — you can see the tradeoff numerically.

## Patterns

**Inverted index structure**: Both strategies use the same nested-dict shape `{field: {value: set(pks)}}` — a classic inverted index. The difference is placement (co-located with documents vs. partitioned by term).

**Partition assignment via hash modulo**: `hash(pk) % num_partitions` for documents, `hash(value) % num_partitions` for terms. Simple, deterministic, no coordination needed.

**Stats accumulator**: Every operation increments counters, enabling `get_stats()` to compute averages. The stats track the DDIA claim: document-partitioned queries touch all partitions; term-partitioned writes touch multiple.

**Async index with explicit flush**: `TermPartitionedDB` supports `async_index=True`, which queues index mutations in `_pending` instead of applying them immediately. This models the real-world approach where global indexes are updated asynchronously to avoid distributed transactions on writes. `flush_index()` drains the queue.

**Index cleanup on update/delete**: Both DB classes carefully remove stale index entries before writing new ones, preventing phantom results from old field values.

## Dependencies

**Imports**: None beyond the standard library (uses only built-in `hash`, `dict`, `set`, `str`, `chr`, `ord`).

**Imported by**:
- `test_secondary_index_partitioning.py` — primary test suite
- `tester_test_secondary_index_partitioning.py` — likely a meta-test or test validator

## Flow

### Write path (document-partitioned)
```
put(pk, fields)
  → hash(pk) → partition_id
  → if pk exists: remove old values from local_index
  → store fields in partition.documents[pk]
  → for each indexed field: add to partition.local_index[field][value]
  → return OperationResult(partitions_touched=1)
```

### Write path (term-partitioned)
```
put(pk, fields)
  → hash(pk) → data_partition_id
  → if pk exists: for each old indexed value:
      → hash(old_value) → index_partition_id (possibly different partition)
      → remove pk from that partition's global_index
  → store fields in data_partition.documents[pk]
  → for each indexed field value:
      → hash(value) → index_partition_id
      → add pk to that partition's global_index
  → return OperationResult(partitions_touched=len(all_touched))
```

### Query path (document-partitioned) — scatter/gather
```
query_by_field(field, value)
  → for EACH partition:
      → check local_index[field][value]
      → collect matching PKs + documents
  → return all results, partitions_touched = num_partitions
```

### Query path (term-partitioned) — targeted
```
query_by_field(field, value)
  → hash(value) → index_partition_id
  → get PKs from that partition's global_index
  → for each PK: hash(pk) → data_partition, fetch document
  → return results, partitions_touched = 1 + distinct_data_partitions
```

## Invariants

1. **PK uniqueness per DB**: A PK maps to exactly one partition (via `hash(pk) % n`). Storing the same PK twice overwrites; it never creates duplicates across partitions.
2. **Index consistency on mutation**: Every `put` and `delete` removes stale index entries before adding new ones. The index is always consistent with document state (unless `async_index=True` and `flush_index` hasn't been called).
3. **Document-partitioned queries always touch all partitions**: `query_by_field` in `DocumentPartitionedDB` unconditionally iterates every partition — there's no short-circuit.
4. **Term partition assignment is deterministic**: Given the same value and `num_partitions`, `_term_partition` always returns the same partition. No rebalancing or dynamic routing.
5. **Empty index cleanup**: When the last PK is removed from an index entry (`set` becomes empty), the entry is deleted from the dict rather than left as an empty set.

## Error Handling

Essentially none. The code trusts its callers:
- No validation on `pk`, `fields`, `partition_id`, `field`, or `value`
- `get` returns `None` for missing documents (via `dict.get`)
- `delete` silently no-ops if the PK doesn't exist
- `query_by_field` returns an empty list if no matches
- `get_partition` will raise `IndexError` on invalid `partition_id` — no guard
- Hash collisions are handled correctly by the set-based index structure, but partition count of 0 would cause `ZeroDivisionError`

This is appropriate for a teaching implementation — errors surface as standard Python exceptions rather than being wrapped.

## Topics to Explore

- [file] `secondary-index-partitioning/test_secondary_index_partitioning.py` — See how the scatter/gather vs. targeted query tradeoff is validated with concrete numbers
- [function] `secondary_index_partitioning.py:compare_strategies` — The pedagogical entry point; run it with varying partition counts and workloads to see the tradeoff shift
- [general] `async-index-consistency` — Explore what happens when you query a term-partitioned DB with `async_index=True` before calling `flush_index` — stale reads are the real-world consequence DDIA warns about
- [file] `range-partitioning/range_partitioning.py` — Compare the range-boundary logic here (`_build_range_boundaries`) with the dedicated range partitioning implementation
- [general] `ddia-chapter-6-partitioning` — Reread DDIA §6.3 "Partitioning and Secondary Indexes" alongside this code to map each concept to its implementation

## Beliefs

- `doc-partitioned-query-touches-all` — `DocumentPartitionedDB.query_by_field` always touches exactly `num_partitions` partitions, regardless of result count (scatter/gather)
- `term-partitioned-write-touches-multiple` — `TermPartitionedDB.put` touches 1 + N partitions where N depends on how many distinct index partitions the document's field values hash to
- `term-partitioned-point-query-is-targeted` — `TermPartitionedDB.query_by_field` looks up a single index partition then fetches documents, avoiding scatter/gather
- `async-index-defers-mutations` — With `async_index=True`, `TermPartitionedDB` queues index operations in `_pending` and only applies them on `flush_index()`, meaning queries can return stale results
- `document-class-unused-by-db` — The `Document` class is defined but never instantiated or required by either DB class; they accept raw `pk`/`fields` pairs

