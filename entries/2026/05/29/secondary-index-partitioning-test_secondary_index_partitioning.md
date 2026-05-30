# File: secondary-index-partitioning/test_secondary_index_partitioning.py

**Date:** 2026-05-29
**Time:** 14:10

I have the test file content from your prompt. Let me explain it based on what the tests reveal about the system.

---

# `test_secondary_index_partitioning.py`

## Purpose

This is the test suite for the `secondary-index-partitioning` module — a reference implementation of the two secondary-index partitioning strategies described in DDIA Chapter 6 (Partitioning): **document-partitioned** (local) indexes and **term-partitioned** (global) indexes. The tests systematically verify the fundamental tradeoff between these strategies: document-partitioned indexes are cheap to write (touch one partition) but expensive to query (scatter/gather across all partitions), while term-partitioned indexes are expensive to write (touch multiple partitions) but cheap to query (touch only the partition owning that term).

The file owns **behavioral verification** of both strategies, covering CRUD, index maintenance, query correctness, async index propagation, range queries, and the comparative benchmarking utility.

## Key Components

The tests import five things from the implementation:

| Import | Role |
|--------|------|
| `Document` | Data model for stored documents (not directly constructed in tests — returned by the DB) |
| `Partition` | Individual partition with `document_count`, `local_index`, `global_index` attributes |
| `OperationResult` | Return type for all DB operations; carries `.data` and `.partitions_touched` |
| `DocumentPartitionedDB` | DB using local (document-partitioned) secondary indexes |
| `TermPartitionedDB` | DB using global (term-partitioned) secondary indexes |

Both DB classes share the same API surface: `put(pk, fields)`, `get(pk)`, `delete(pk)`, `query_by_field(field, value)`, and `get_partition(i)`. `TermPartitionedDB` additionally supports `flush_index()` (for async mode) and `query_range()` (for range-partitioned terms).

### Test Classes (15 total, in deliberate pedagogical order)

1. **`TestBasicCRUD`** — put/get/delete/update on both DB types
2. **`TestHashPartitioning`** — verifies documents distribute across partitions (not all land on one)
3. **`TestDocPartitionedIndex`** — inspects `Partition.local_index` directly after insert/update/delete
4. **`TestDocPartitionedQuery`** — scatter/gather: queries always touch all 4 partitions
5. **`TestTermPartitionedIndex`** — inspects `Partition.global_index` to verify term→partition routing
6. **`TestTermPartitionedQuery`** — queries touch ≤4 partitions (typically 1)
7. **`TestWriteCost`** — document writes touch 1 partition; term writes touch ≥1
8. **`TestUpdateIndex`** — old index entries are cleaned up on update (both strategies)
9. **`TestDeleteCleanup`** — index entries are cleaned up on delete (both strategies)
10. **`TestAsyncIndex`** — term-partitioned async mode: stale before flush, correct after
11. **`TestRangeQuery`** — range-partitioned terms with `query_range("color", "a", "d")`
12. **`TestCompareStrategies`** — `compare_strategies()` utility returns avg write/query partition counts
13. **`TestManyDocuments`** — 1000-document scale test across 5 colors; both strategies agree
14. **`TestEdgeCases`** — missing fields, nonexistent values, updates to non-indexed fields
15. **`TestExampleUsage`** — integration test matching the module's example usage

## Patterns

**Tradeoff-centric test design.** The tests are structured to demonstrate the DDIA tradeoff, not just verify correctness. `TestWriteCost` proves document writes touch exactly 1 partition (`assert r.partitions_touched == 1`) while term writes touch more. `TestDocPartitionedQuery` proves scatter/gather (`assert result.partitions_touched == 4`). `TestTermPartitionedQuery` proves targeted queries (`assert result.partitions_touched <= 4`). These assertions *are* the lesson.

**White-box partition inspection.** Tests like `TestDocPartitionedIndex` and `TestTermPartitionedIndex` reach into `Partition.local_index` and `Partition.global_index` directly via `db.get_partition(hash(pk) % 4)`. This verifies index placement, not just query results.

**Deterministic hashing.** Tests rely on Python's `hash()` being deterministic within a process to compute expected partition IDs (`pid = hash("p1") % 4`). This lets tests assert *which* partition holds a document or index entry.

**Constructor-controlled behavior.** The DB classes accept configuration through constructor args:
- `num_partitions` (always 4 in tests)
- `indexed_fields` (list of field names like `["color", "size"]`)
- `async_index=True` (term-partitioned only)
- `partition_by="range"` (term-partitioned only)

**`OperationResult` as instrumentation.** Every operation returns an `OperationResult` with `.partitions_touched`, making the partition-access cost observable and testable — the key metric for comparing strategies.

## Dependencies

**Imports:**
- `pytest` — test framework (imported but no fixtures or parametrize used; tests are plain class methods)
- `secondary_index_partitioning` — the implementation module

**Imported by:**
- `tester_test_secondary_index_partitioning.py` — likely a meta-test or test runner wrapper (following the repo's `tester_test_*` convention)

## Flow

Each test class follows the same pattern:

1. Instantiate a `DocumentPartitionedDB` or `TermPartitionedDB` with 4 partitions and specified indexed fields
2. Insert documents via `put(pk, fields_dict)`
3. Assert on one of:
   - **Data correctness**: `result.data` matches expected content
   - **Partition cost**: `result.partitions_touched` matches expected count
   - **Index state**: partition's `local_index` or `global_index` contains/excludes expected entries

The `TestCompareStrategies` test exercises a higher-level `compare_strategies()` function that inserts the same data into both DB types, runs the same queries, and returns aggregate statistics (`avg_partitions_per_write`, `avg_partitions_per_query`, `summary`).

The `TestAsyncIndex` class tests a two-phase flow: writes queue index updates instead of applying them, then `flush_index()` applies the pending updates. This models real-world async global index propagation.

## Invariants

- **Document-partitioned writes always touch exactly 1 partition** — enforced by `TestWriteCost.test_doc_write_touches_one` and `TestExampleUsage`
- **Document-partitioned queries always touch all partitions** (scatter/gather) — enforced by `TestDocPartitionedQuery.test_scatter_gather` (`partitions_touched == 4`)
- **Term-partitioned queries touch ≤ N partitions** — enforced by `TestTermPartitionedQuery` (`partitions_touched <= 4`)
- **Index cleanup on update and delete** — both strategies must remove stale index entries (4 tests across `TestUpdateIndex` and `TestDeleteCleanup`)
- **Async mode is eventually consistent** — queries return stale results before `flush_index()`, correct results after
- **Both strategies return identical query results** — `TestManyDocuments` asserts `len(r1.data) == len(r2.data) == 200` for every color
- **Documents with missing indexed fields are stored but not indexed** — `TestEdgeCases.test_missing_indexed_field`

## Error Handling

The tests don't exercise error paths (no `pytest.raises` usage). The code under test appears to handle edge cases gracefully rather than raising:
- `get("missing")` returns `OperationResult(data=None)` rather than raising `KeyError`
- `query_by_field("color", "purple")` returns empty results rather than raising
- Documents missing indexed fields are stored without index entries — no validation error

---

## Topics to Explore

- [file] `secondary-index-partitioning/secondary_index_partitioning.py` — The implementation of both partitioning strategies, `Partition` internals, and the `compare_strategies` utility
- [function] `secondary-index-partitioning/secondary_index_partitioning.py:TermPartitionedDB.flush_index` — How async index updates are queued and applied, modeling real-world eventual consistency of global indexes
- [function] `secondary-index-partitioning/secondary_index_partitioning.py:TermPartitionedDB.query_range` — Range query implementation over range-partitioned terms — how partition boundaries are determined
- [general] `document-vs-term-partitioned-tradeoff` — DDIA Chapter 6's analysis of why document-partitioned indexes favor writes and term-partitioned indexes favor reads, and how this maps to real systems (Elasticsearch uses document-partitioned; DynamoDB GSIs are term-partitioned)
- [file] `secondary-index-partitioning/tester_test_secondary_index_partitioning.py` — Meta-test wrapper that likely validates the test suite itself or provides additional test infrastructure

## Beliefs

- `doc-partitioned-write-touches-one` — `DocumentPartitionedDB.put()` always touches exactly 1 partition regardless of how many fields are indexed, because the index is co-located with the document
- `doc-partitioned-query-scatters-all` — `DocumentPartitionedDB.query_by_field()` always touches all N partitions (scatter/gather) because each partition maintains only a local index of its own documents
- `term-partitioned-async-is-eventually-consistent` — `TermPartitionedDB` with `async_index=True` returns stale query results until `flush_index()` is called, modeling the real-world propagation delay of global secondary indexes
- `both-strategies-clean-indexes-on-mutation` — Both `DocumentPartitionedDB` and `TermPartitionedDB` remove stale index entries when a document is updated or deleted, preventing phantom results
- `operation-result-tracks-partition-cost` — Every DB operation returns an `OperationResult` with a `partitions_touched` field, making the read/write amplification cost observable and comparable between strategies

