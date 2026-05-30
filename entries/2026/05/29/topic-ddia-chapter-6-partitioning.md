# Topic: DDIA Chapter 6 covers the theory behind partition strategies, rebalancing, and request routing that this code implements

**Date:** 2026-05-29
**Time:** 14:00

# DDIA Chapter 6: Partitioning — From Theory to Code

Chapter 6 of *Designing Data-Intensive Applications* lays out three core problems: **how to partition data**, **how to rebalance when nodes change**, and **how to route requests to the right partition**. This codebase implements all three, each in a separate module that isolates one strategy.

---

## Partition Strategies

### Range Partitioning

`range-partitioning/range_partitioning.py` implements what Kleppmann calls **key-range partitioning**. Each `Partition` owns a half-open interval `[start_key, end_key)` (line 13), and `RangePartitionedStore` maintains a sorted `_boundaries` list (line 108) so that `_find_partition_index` can route any key via `bisect.bisect_right` in O(log n) time (lines 110–112).

The advantage Kleppmann highlights — efficient range scans — is directly visible in `range_scan` (lines 137–150), which identifies the first and last relevant partitions and concatenates their results. A hash-based scheme couldn't do this without touching every partition.

### Hash Partitioning

`consistent-hashing/consistent_hashing.py` implements **hash partitioning** using a consistent hash ring with virtual nodes. Keys are hashed to a 32-bit position via MD5 (line 11), and `get_node` (lines 78–84) walks clockwise to the next virtual node. This is the approach Kleppmann describes as avoiding hotspots by distributing keys uniformly — at the cost of losing key ordering entirely (no range scan method exists on this class).

The `num_vnodes` parameter (default 150, line 19) addresses the load imbalance problem Kleppmann warns about: with too few tokens per node, some nodes get disproportionately large arcs. The `load_imbalance` method (lines 147–153) directly measures this as `max_load / avg_load`.

### Secondary Index Partitioning

`secondary-index-partitioning/secondary_index_partitioning.py` implements both approaches Kleppmann describes for partitioning secondary indexes:

- **Document-partitioned (local) indexes** in `DocumentPartitionedDB`: each partition maintains its own index covering only its local documents. A `query_by_field` call (lines 103–108) must **scatter-gather across all partitions** — exactly the fan-out cost Kleppmann warns about. The stats tracking (`total_partitions_touched_queries`, line 108) makes this cost explicit: every query touches `num_partitions` partitions.

- **Term-partitioned (global) indexes** in `TermPartitionedDB`: the index itself is partitioned by term value. `_term_partition` (lines 171–177) hashes or range-partitions the index term, so a query hits exactly one partition. The tradeoff is writes: storing a document may require updating index entries on multiple partitions. The `async_index` flag (line 155) models the async update approach Kleppmann describes for eventual consistency of global indexes.

---

## Rebalancing

### Dynamic Splitting and Merging

Range partitioning uses **dynamic rebalancing** via split and merge — the strategy Kleppmann associates with HBase and RethinkDB. When a partition exceeds `max_partition_size`, `put` automatically splits it at the median key (lines 118–121 of `range_partitioning.py`). The `Partition.split` method (lines 69–79) creates a new right partition and adjusts boundaries. Conversely, `merge_small_partitions` (lines 152–166) recombines adjacent undersized partitions.

### Consistent Hashing Rebalancing

When a node is added or removed from the hash ring, `add_node` (lines 27–46) and `remove_node` (lines 48–66) return transfer maps: `{(arc_start, arc_end): (from_node, to_node)}`. This models what Kleppmann describes as the key advantage of consistent hashing — only keys in the affected arcs move, not the entire dataset. The `weight` parameter (line 27) supports heterogeneous nodes by scaling virtual node count.

### Consumer Group Rebalancing

`partitioned-log/partitioned_log.py` implements Kafka-style consumer group rebalancing (line 317: `rebalance`). When consumers join or leave a group (lines 305, 309), partitions are redistributed. The `Producer` class (lines 144–175) handles the routing side: keyed messages hash to a fixed partition (line 161), while unkeyed messages round-robin (lines 163–165) — exactly the two strategies Kleppmann describes for Kafka producers.

---

## Request Routing

The code demonstrates what Kleppmann calls the **partition-aware client** approach. There's no separate routing tier; instead, the store itself maintains routing metadata:

- In `RangePartitionedStore`, the `_boundaries` list acts as a partition map. `_find_partition_index` (lines 110–112) is the routing function — a binary search that resolves any key to its owning partition in O(log n).
- In `ConsistentHashRing`, `get_node` (lines 78–84) is the routing function — walk clockwise from the key's hash position to find the owning node.
- In `DocumentPartitionedDB`, primary key lookups route to exactly one partition via `_partition_for` (line 57: `hash(pk) % num_partitions`), while secondary index queries must fan out.

The `get_nodes` method in consistent hashing (lines 86–101) extends routing to support **replication**: it walks the ring collecting `replication_factor` distinct physical nodes, implementing the preference list concept from Dynamo that Kleppmann describes.

---

## Topics to Explore

- [file] `secondary-index-partitioning/secondary_index_partitioning.py` — Read the full `TermPartitionedDB` implementation (lines 200+) to see how writes fan out to multiple index partitions and how the async index update queue works
- [function] `partitioned-log/partitioned_log.py:rebalance` — The consumer group rebalancing algorithm at line 317; compare with Kafka's cooperative sticky assignor
- [function] `consistent-hashing/consistent_hashing.py:get_load_distribution` — Measures actual arc ownership per node (lines 117–131); useful for understanding why 150 vnodes is the default and how weight affects balance
- [file] `hinted-handoff/hinted_handoff.py` — The coordinator at line 72 routes requests and handles node failures, connecting partitioning to fault tolerance (Chapter 5 meets Chapter 6)
- [general] `scatter-gather-cost` — Compare `DocumentPartitionedDB.query_by_field` (touches all partitions) vs `TermPartitionedDB` query (touches one) with concrete metrics from `get_stats()`

## Beliefs

- `range-partition-split-at-median` — `Partition.split` always divides at the median key (index `len // 2`), which can produce uneven partitions when key distribution is skewed
- `consistent-hash-ring-uses-md5` — All ring positions are computed via MD5 truncated to 32 bits; the hash function is not pluggable
- `document-partitioned-query-touches-all` — `DocumentPartitionedDB.query_by_field` always scans every partition regardless of selectivity, and records `num_partitions` as partitions touched
- `term-partitioned-write-fans-out` — A single `TermPartitionedDB.put` may update global index entries on multiple partitions (one per indexed field value), making writes more expensive than in document-partitioned mode
- `range-partition-routing-is-log-n` — `RangePartitionedStore._find_partition_index` uses `bisect.bisect_right` on the sorted boundary list, giving O(log n) key-to-partition resolution

