# Topic: This implementation has no snapshotting or log truncation — consider how the log grows unboundedly and what a compaction extension would require

**Date:** 2026-05-29
**Time:** 13:52

I have enough from the observations to give a thorough explanation. Let me work with what's already gathered.

---

## Unbounded Log Growth in the Raft Implementation

### The Core Problem

The Raft log in `raft-consensus/raft.py` is an in-memory Python list (`self._log`) that only ever grows. Every client request appends a `LogEntry` (line 30), and nothing ever removes old entries. There is no snapshotting, no log truncation, and no compaction — a grep for `snapshot|truncat|compact|trim` across the Raft module returned zero matches.

### Where Entries Accumulate

The log starts with a single sentinel entry at index 0 (line 30):

```python
self._log = [LogEntry(term=0, index=0, command=None)]  # sentinel
```

It grows in two places:

1. **`client_request`** (line ~163): the leader appends a new entry for every client command:
   ```python
   entry = LogEntry(term=self._current_term, index=self._last_log_index() + 1, command=command)
   self._log.append(entry)
   ```

2. **`handle_append_entries`** (lines ~136–155): followers append entries received from the leader during replication. Entries are either inserted at the correct position or appended at the tail.

Neither path ever removes committed entries. The `_commit_index` (line 33) tracks how far the log has been committed, and `_last_applied` (line 34) tracks how far it has been applied to the state machine — but advancing these pointers doesn't free the entries behind them. Every entry ever committed remains in `self._log` forever.

### Why This Matters

In a real system, the log serves two purposes: (1) replication of uncommitted entries, and (2) recovery after a crash. Once an entry is committed and applied to the state machine, the log entry is only needed to bring slow followers up to date. Without truncation:

- **Memory grows linearly** with the number of client requests. In a long-running system, this is an OOM risk.
- **Leader restart is expensive**: the leader must replay the entire log to reconstruct state.
- **Slow follower catch-up is expensive**: `_make_append_entries` (line ~182) sends `self._log[ni:]` — the entire suffix from the follower's `next_index` forward. If a follower falls far behind, this slice could be enormous, but at least it's bounded by the log length. The real problem is that `ni` could be 1 while the log has millions of entries.

### What a Compaction Extension Would Require

Adding snapshotting and log truncation to this implementation would touch several areas:

#### 1. Snapshot Creation

You'd need a way to serialize the state machine at a given `last_applied` index. The current code has no explicit state machine — `_last_applied` is tracked but never used to apply commands. A snapshot would capture:
- The state machine's data at `last_applied`
- The term and index of the last included entry
- The current cluster configuration

#### 2. Log Truncation

After a snapshot at index `N`, entries `0..N` can be discarded. This requires changing the sentinel assumption: `self._log[prev_log_index]` (line ~140) does a direct array index, which assumes index 0 is always present. After truncation, you'd need an offset — `self._log[prev_log_index - snapshot_last_index]` — or switch to a dict/deque. Every place that indexes into `self._log` by log index would need adjustment:

- `_last_log_index()` / `_last_log_term()` (lines 68–72)
- `handle_append_entries` log consistency check (line ~140)
- `_make_append_entries` entry slicing (line ~186)
- `_is_log_up_to_date` (line ~96)

#### 3. InstallSnapshot RPC

When a leader has already truncated entries that a slow follower needs, it can't send them via `AppendEntries`. Raft's solution is a separate `InstallSnapshot` RPC. This implementation would need:
- A new message type alongside `request_vote` and `append_entries`
- Leader logic to detect when `next_index[peer]` falls below the snapshot boundary
- Follower logic to accept and install the snapshot, resetting its log

#### 4. Contrast with the WAL Module

The `write-ahead-log/wal.py` module shows what truncation looks like in this same codebase. It has an explicit `truncate(up_to_seq)` method (line ~178) that removes records below a sequence number, and file rotation via `_rotate()` (line ~107) and `_maybe_rotate()` (line ~134). The LSM tree module (`log-structured-merge-tree/lsm.py`) goes further with a full `compact()` method (line 319) that merges SSTables and removes tombstones. These serve as reference patterns for what the Raft module is missing.

The key difference: those modules manage on-disk structures with explicit lifecycle management. The Raft log is purely in-memory with no lifecycle at all — making unbounded growth both simpler to trigger and simpler to fix (no file management), but requiring the additional InstallSnapshot protocol complexity that the WAL doesn't need.

---

## Topics to Explore

- [function] `raft-consensus/raft.py:_make_append_entries` — See how the full log suffix is sliced for replication; this is the hot path that snapshots would optimize
- [function] `raft-consensus/raft.py:handle_append_entries` — The log consistency check indexes directly into `self._log`, which would break after truncation
- [file] `write-ahead-log/wal.py` — Compare its `truncate()` and `_rotate()` methods as patterns for log lifecycle management
- [file] `log-structured-merge-tree/lsm.py` — The `compact()` method shows a complete compaction implementation with tombstone removal in this same codebase
- [general] `raft-installsnapshot-rpc` — The Raft paper's Section 7 describes the InstallSnapshot RPC that would complement log truncation

---

## Beliefs

- `raft-log-never-shrinks` — `self._log` in `RaftNode` is append-only: no method ever removes entries, so it grows monotonically with client requests
- `raft-log-direct-indexed` — Log entries are accessed by log index as a direct array index (`self._log[prev_log_index]`), which assumes all entries from index 0 onward are present in memory
- `raft-no-state-machine-apply` — `_last_applied` is tracked but never used to actually apply commands to a state machine, making snapshotting impossible without first implementing application logic
- `wal-has-truncation-raft-does-not` — The WAL module provides explicit `truncate(up_to_seq)` and file rotation; the Raft module has no equivalent, despite both being append-only log abstractions
- `raft-sentinel-entry-assumption` — The log is initialized with a sentinel `LogEntry(term=0, index=0)` at position 0, and multiple methods depend on this entry always being present at `self._log[0]`

