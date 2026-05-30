# File: partitioned-log/test_partitioned_log.py

**Date:** 2026-05-29
**Time:** 13:39

I'll work from the test file content you provided, which reveals the implementation's API surface thoroughly.

---

## Purpose

`test_partitioned_log.py` is the **comprehensive behavioral specification** for a Kafka-style partitioned append-only log. It validates the `partitioned_log` module — a reference implementation of the log-based messaging system described in DDIA Chapter 11 (Stream Processing). The file owns verification of the entire produce-consume lifecycle: topic creation, message routing, consumer group coordination, offset management, retention/compaction, and disk persistence.

It serves double duty: both as a regression suite and as **executable documentation** of how the Kafka model works, making it a teaching artifact for the DDIA implementations project.

## Key Components

The file contains 14 test functions, each targeting a specific Kafka concept. No classes or constants are defined — all state is local to each test.

### Core API under test

| Class | Role |
|---|---|
| `Broker` | Central message store; manages topics, retention, compaction, consumer groups, persistence |
| `Producer` | Writes messages to topics; handles partitioning (key-hash or round-robin) |
| `Consumer` | Reads messages from assigned partitions; manages offsets, group membership |
| `Topic` | Partition container; exposes `partition_size()`, `earliest_offset()`, `latest_offset()`, `read()` |
| `Message` | Consumed record with `.value`, `.offset`, `.key` fields |
| `RecordMetadata` | Returned from `send()`; carries `.topic`, `.partition`, `.offset` |

### Test inventory

| Test | Kafka concept |
|---|---|
| `test_basic_produce_consume` | Produce → poll → verify ordering and content; also tests idempotent empty poll |
| `test_key_partitioning` | Deterministic key-based hash partitioning |
| `test_round_robin` | Null-key round-robin distribution |
| `test_consumer_groups` | Group-based partition assignment (disjoint, complete coverage) |
| `test_rebalancing` | Dynamic rebalance on join/leave |
| `test_independent_groups` | Multi-group fan-out (pub-sub semantics) |
| `test_offset_commit_restore` | Offset persistence across consumer instances |
| `test_seek_replay` | Manual offset reset for replay |
| `test_auto_commit_and_offset_reset` | `auto_commit=True` and `auto_offset_reset='latest'` |
| `test_retention_and_compaction` | Size-based retention truncation + key-based log compaction |
| `test_batch_produce_and_multiple_topics` | `send_batch()` and cross-topic isolation |
| `test_more_consumers_than_partitions` | Idle consumers when group size > partition count |
| `test_read_and_append_after_compaction` | Offset gaps from compaction: skip-reads, append correctness, consumer traversal |
| `test_disk_persistence` | Broker serialization/deserialization from filesystem |

## Patterns

**Self-contained setup per test.** Every test creates its own `Broker`, topics, producers, and consumers. No shared fixtures, no `setUp`/`tearDown` (except `test_disk_persistence` which uses `tempfile`/`shutil` for cleanup). This makes each test independently runnable and debuggable.

**Assert-and-print.** Each test ends with `print("PASS: ...")`. Combined with the `__main__` runner at the bottom, this lets the suite run outside pytest with visible per-test feedback — useful for the DDIA project's teaching context.

**Contract-driven testing.** Tests verify API contracts (`isinstance(meta1, RecordMetadata)`), not implementation internals. The tests never reach into private fields or mock anything — they exercise the public API end-to-end.

**Composite tests.** Several tests verify two related concepts together (e.g., `test_auto_commit_and_offset_reset` covers tests 9+10, `test_retention_and_compaction` covers tests 11+12). The comments referencing "Test N" suggest the tests were originally designed from a numbered specification.

## Dependencies

**Imports:**
- `shutil`, `tempfile` — standard library, used only by `test_disk_persistence` for temp directory lifecycle
- `partitioned_log` — the implementation module; imports `Broker`, `Producer`, `Consumer`, `Message`, `RecordMetadata`, `Topic`

**Imported by:**
- Nothing imports this file. It's a leaf test module. `test_smoke.py` in the same directory likely runs a subset of these tests or a simpler smoke check.

## Flow

Each test follows the same pattern:

1. **Arrange**: Create a `Broker`, create topic(s) with specific partition counts, instantiate `Producer`/`Consumer`
2. **Act**: Produce messages (with explicit partition, key-based routing, or round-robin), then poll via consumers
3. **Assert**: Verify message content, ordering, offsets, partition assignments, or state changes

The `__main__` block runs all 14 tests sequentially and prints a summary. Any assertion failure halts execution at that test.

### Notable control flows:

- **`test_rebalancing`**: Subscribes `c1` alone (gets 4 partitions), adds `c2` (triggers rebalance to 2+2), closes `c2` (c1 recovers all 4). This tests that `subscribe()` and `close()` have immediate rebalancing side-effects.
- **`test_offset_commit_restore`**: Uses `max_messages=3` to partially consume, commits, closes the consumer, then creates a new consumer in the same group to verify resume-from-committed-offset.
- **`test_read_and_append_after_compaction`**: The most intricate test — verifies that after compaction removes offset 1, reading from offset 1 skips forward to offset 2, and that new appends get offset 4 (not 3).

## Invariants

The tests collectively enforce these invariants about the system:

1. **Offsets are monotonically increasing per partition.** `meta1.offset == 0`, `meta2.offset == 1`, and post-compaction appends continue from `latest_offset`.
2. **Consumer group partition assignment is a complete, disjoint cover.** `len(c1.assignment) + len(c2.assignment) == num_partitions` and the sets are disjoint.
3. **Key-based partitioning is deterministic.** Same key always maps to same partition across multiple sends.
4. **Round-robin distributes evenly.** 9 messages across 3 partitions yields `[3, 3, 3]`.
5. **Committed offsets survive consumer replacement.** A new consumer in the same group resumes from the last commit point.
6. **Compaction preserves only the latest value per key** and retains all null-key messages.
7. **Retention removes from the front of the log.** After enforcing retention with `max_log_size=5` on 10 messages, `earliest_offset` becomes 5.
8. **Idle consumers are possible.** With 1 partition and 3 consumers, exactly 2 are idle (empty assignment).
9. **Independent consumer groups see the full log independently.** Two groups reading the same topic both get all messages.
10. **Poll is non-destructive but advances position.** A second `poll()` with no new messages returns empty.

## Error Handling

The test file contains **no error handling** — no `try/except`, no expected-exception tests. All assertions are positive (the happy path). The only resource cleanup is in `test_disk_persistence`, which uses a `try/finally` block to ensure `shutil.rmtree(tmpdir)` runs regardless of test outcome.

Notably absent: tests for producing to nonexistent topics, consuming from unassigned partitions, invalid seek offsets, or duplicate consumer IDs. This is a specification of correct behavior, not a robustness suite.

## Topics to Explore

- [file] `partitioned-log/partitioned_log.py` — The implementation behind all these tests; particularly how `Broker.compact()` handles offset gaps and how consumer group rebalancing is triggered synchronously
- [function] `partitioned-log/partitioned_log.py:Topic.read` — How it skips over compaction-created offset gaps when reading from a removed offset
- [file] `partitioned-log/test_smoke.py` — Likely a minimal subset; compare what it covers vs this full suite
- [general] `consumer-group-rebalancing` — The rebalancing strategy (range vs round-robin assignment) and whether `subscribe()` triggers it eagerly or lazily
- [general] `compaction-offset-semantics` — How the log maintains logical offset continuity after physical message removal, and whether `earliest_offset` reflects the first *surviving* message or the original first message

## Beliefs

- `partitioned-log-poll-advances-position` — `Consumer.poll()` advances the consumer's internal offset so a second poll with no new messages returns an empty list
- `partitioned-log-key-hash-is-deterministic` — The same key always hashes to the same partition across multiple `Producer.send()` calls, regardless of timing
- `partitioned-log-group-assignment-is-disjoint-cover` — Consumer group partition assignment always forms a complete, non-overlapping partition of all topic partitions across group members
- `partitioned-log-compaction-preserves-null-keys` — Log compaction retains all messages without a key and only deduplicates keyed messages, keeping the latest value per key
- `partitioned-log-offsets-survive-compaction` — After compaction removes messages, new appends continue from `latest_offset` (not from the count of surviving messages), preserving offset monotonicity

