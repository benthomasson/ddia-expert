# File: partitioned-log/partitioned_log.py

**Date:** 2026-05-29
**Time:** 09:01

# `partitioned-log/partitioned_log.py`

## Purpose

This file implements a **Kafka-style partitioned append-only log** — the core abstraction from DDIA Chapter 11 (Stream Processing). It models the key components of a message broker: topics split into partitions, producers that write messages with key-based or round-robin routing, consumers that track read positions via offsets, and consumer groups that coordinate partition assignment across members. The `Broker` class ties it all together, adding persistence, retention enforcement, and log compaction.

This is a teaching implementation — everything runs in-process with Python lists as the storage backend, no networking, no replication. The goal is to demonstrate the *semantics* of a partitioned log, not production performance.

## Key Components

### Data Classes

**`Message`** — The unit of data in the log. Carries a `key` (used for partition routing and compaction), `value` (the payload), `timestamp`, and optional `headers`. The `topic`, `partition`, and `offset` fields are set by `Topic.append` at write time — they're `None` on construction and populated as metadata once the message lands in a partition.

**`RecordMetadata`** — The acknowledgment returned to a producer after a successful write. Contains the coordinates needed to locate the message: topic, partition, offset, timestamp.

### `Topic`

Owns the partitioned storage for a single named topic. Each partition is a `list[Message]` with a separate `_base_offsets` tracker that advances when messages are truncated or compacted away.

- **`append(partition, message)`** — Assigns the next offset (last offset + 1, or base offset if empty), stamps the message with topic/partition/offset metadata, appends to the partition list. Returns the assigned offset.
- **`read(partition, offset, max_messages)`** — Linear scan from the requested offset forward. If the requested offset is below the base (already truncated), it clamps up to the base. Returns up to `max_messages` messages.
- **`truncate(partition, count)`** — Removes `count` messages from the head of a partition. Advances `_base_offsets` so future offset calculations remain correct.
- **`compact_partition(partition)`** — Log compaction: for each key, retains only the *last* occurrence. Keyless messages are always kept. Updates the base offset to match the first retained message.
- **`earliest_offset` / `latest_offset`** — Boundary queries. `latest_offset` returns one past the last message (the next offset that *would* be assigned), matching Kafka's convention.

### `Producer`

Routes messages to partitions and writes them through the broker.

**Partitioning strategy** in `send()`:
1. **Explicit partition** — caller specifies `partition=N`
2. **Key-based** — MD5 hash of the key mod partition count (deterministic routing, same key always hits the same partition)
3. **Round-robin** — a per-topic counter cycles through partitions for keyless messages

`send_batch()` is a convenience that iterates over a list of dicts, calling `send()` for each. `flush()` is a no-op stub (there's no buffering to flush).

### `Consumer`

Tracks read position per assigned `(topic, partition)` pair.

- **`subscribe(topics)`** — If the consumer has a `group_id`, it registers with the consumer group and triggers a rebalance. Without a group, it self-assigns all partitions of the subscribed topics.
- **`_init_offsets()`** — Resolves the starting offset for each assigned partition: first checks for a committed offset (group-mode), then falls back to `auto_offset_reset` ("earliest" or "latest").
- **`poll(max_messages)`** — Reads from each assigned partition in order, advancing the local offset cursor. If `auto_commit` is enabled, commits after every non-empty poll.
- **`seek(topic, partition, offset)`** — Manual offset reset, clamped to the earliest available offset.
- **`commit()`** — Persists current offsets to the broker's `_committed_offsets` map (keyed by group, topic, partition).
- **`close()`** — Removes this consumer from its group, triggering rebalance.

### `ConsumerGroup`

Manages **partition assignment across consumers** using round-robin rebalancing.

`rebalance()` collects all partitions from all subscribed topics, sorts consumers by ID, and distributes partitions round-robin. This fires on every `add_consumer` or `remove_consumer` call. After reassignment, each consumer's offsets are re-initialized.

### `Broker`

Central coordinator. Owns topic registry, consumer groups, committed offsets, and optional disk persistence.

- **Topic management** — `create_topic`, `get_topic`, `list_topics`
- **`compact(topic)`** — Runs `compact_partition` on every partition, returns removal stats.
- **`enforce_retention(topic)`** — Two retention policies, applied per-partition:
  - *Size-based*: if partition exceeds `max_log_size`, truncates the excess from the head.
  - *Time-based*: if `retention_ms > 0`, truncates messages older than the cutoff.
- **Persistence** — When `persist_dir` is set:
  - Messages are appended as JSONL files (`{topic}_{partition}.jsonl`).
  - Committed offsets are written as a single `offsets.json`.
  - On startup, `_load_from_disk` reconstructs topics and replays messages from JSONL files.

## Patterns

- **Kafka's offset model**: Offsets are partition-local, monotonically increasing integers. The `_base_offsets` array tracks the logical start after truncation/compaction, so the offset space never resets.
- **Separation of routing and storage**: The `Producer` decides *which* partition; the `Topic` handles append and offset assignment. This mirrors Kafka's client-side partitioning.
- **Eager rebalance**: Consumer group rebalance is synchronous and happens on every membership change — no incremental/cooperative protocol. This is the simplest correct approach.
- **Dual retention**: Size-based and time-based retention can be configured independently per topic, matching Kafka's `log.retention.bytes` and `log.retention.ms`.

## Dependencies

**Imports**: All stdlib — `hashlib` (key hashing), `json`/`os` (persistence), `time` (timestamps), `uuid` (consumer IDs), `dataclasses`, `typing`.

**Imported by**: `test_partitioned_log.py` and `test_smoke.py` — the module has no downstream production consumers, it's a standalone reference implementation.

## Flow

**Write path**: `Producer.send()` → resolve partition → construct `Message` → `Topic.append()` (assigns offset, stamps metadata) → `Broker._persist_message()` (append JSONL if persistence enabled) → return `RecordMetadata`.

**Read path**: `Consumer.poll()` → iterate assigned `(topic, partition)` pairs → `Topic.read()` (linear scan from current offset) → advance consumer's offset cursor → optionally auto-commit → return messages.

**Group coordination**: `Consumer.subscribe()` → `ConsumerGroup.add_consumer()` → `rebalance()` (round-robin assignment) → `Consumer._init_offsets()` (resolve committed or reset offset per partition).

**Retention/compaction**: `Broker.enforce_retention()` iterates partitions, truncates by size then by time. `Broker.compact()` deduplicates by key within each partition. Both advance `_base_offsets` to maintain offset continuity.

## Invariants

- **Partition count**: Must be between 1 and 128 (enforced in `Topic.__init__`).
- **Offset monotonicity**: Offsets within a partition are strictly increasing. `_base_offsets` only increases (via `truncate` and `compact_partition`).
- **Offset clamping**: Both `Consumer.poll()` and `Consumer.seek()` clamp offsets to `earliest_offset` — a consumer can never read truncated data or get stuck at an invalid position.
- **Compaction preserves keyless messages**: Messages with `key=None` are always retained during compaction.
- **Committed offsets are scoped to (group, topic, partition)**: A consumer without a `group_id` can call `commit()` but only persists to the broker map for group-based consumers.
- **Round-robin is per-topic**: The producer's `_rr_counters` dict tracks a separate counter for each topic.

## Error Handling

Minimal and fail-fast:
- `Topic.__init__` raises `ValueError` for invalid partition counts.
- `Broker.get_topic` raises `KeyError` for unknown topics — this propagates up through `Producer.send()` and `Consumer.poll()`.
- No try/except blocks anywhere — persistence I/O failures, invalid JSON, and file-not-found during `_load_from_disk` all propagate as unhandled exceptions.
- No validation on partition index bounds in `Topic.append`/`read` — an out-of-range partition will raise `IndexError` from the list access.

## Topics to Explore

- [file] `partitioned-log/test_partitioned_log.py` — Full test suite covering consumer groups, retention, compaction, and persistence — shows the intended usage patterns
- [function] `partitioned-log/partitioned_log.py:ConsumerGroup.rebalance` — Compare this eager round-robin approach with Kafka's range/sticky assignors and cooperative rebalance protocol
- [general] `log-compaction-vs-retention` — How compaction (keep latest per key) and retention (time/size truncation) interact — they can run independently but both mutate `_base_offsets`
- [file] `change-data-capture/cdc.py` — CDC systems are a primary consumer of partitioned logs; see how this pattern connects to the CDC implementation in the same repo
- [general] `kafka-consumer-offset-semantics` — DDIA Ch. 11 discussion of exactly-once vs at-least-once delivery — this implementation's `auto_commit` in `poll()` creates at-least-once semantics with potential duplicate processing on crash

## Beliefs

- `partitioned-log-offset-monotonic` — Offsets within a partition are strictly increasing and never reused; `_base_offsets` only advances forward via `truncate` and `compact_partition`
- `partitioned-log-key-routing-deterministic` — Messages with the same key always route to the same partition (MD5 hash mod partition count), guaranteeing per-key ordering
- `partitioned-log-compaction-preserves-keyless` — Log compaction retains all messages with `key=None`; only keyed messages are deduplicated to their latest occurrence
- `partitioned-log-no-input-validation-on-partition-index` — `Topic.append` and `Topic.read` do not bounds-check the partition parameter; out-of-range values raise `IndexError` from list access
- `partitioned-log-rebalance-is-eager-full` — Every consumer group membership change triggers a full reassignment of all partitions across all consumers via round-robin, with no incremental or cooperative protocol

