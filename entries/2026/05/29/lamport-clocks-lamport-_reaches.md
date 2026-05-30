# Function: _reaches in lamport-clocks/lamport.py

**Date:** 2026-05-29
**Time:** 13:15

## `_reaches(source, target)` — Causal Reachability via BFS

### Purpose

`_reaches` determines whether one event **causally precedes** another in a distributed system's event graph. It implements the core reachability check that backs the `happens_before` relation — Lamport's "→" operator. Without this function, you can compare timestamps, but timestamps alone can't distinguish causally related events from concurrent ones (that's the whole point of Lamport's formalism).

### Contract

**Preconditions:**
- Both `source` and `target` are `Event` instances belonging to the same distributed system (i.e., connected through `_parent` / `_cause` links).
- The event graph is a DAG — no cycles. This is guaranteed by construction: `_parent` always points to a strictly earlier event on the same node, and `_cause` points to a send event on a different node that must have occurred before the receive.

**Postconditions:**
- Returns `True` if and only if there exists a directed path from `target` back to `source` through `_parent` and `_cause` edges.
- Returns `False` if no such path exists (events are concurrent or `source` is after `target`).

**Invariant:** The function never modifies the event graph.

### Parameters

| Parameter | Type | Meaning |
|-----------|------|---------|
| `source` | `Event` | The candidate predecessor — "did this happen first?" |
| `target` | `Event` | The candidate successor — we walk backward from here |

Edge cases:
- If `source is target`, returns `True` on the first iteration (identity check passes immediately). The caller `happens_before` guards against this by checking `event_a is event_b` beforehand and returning `None`.
- If either event has no parents or causes (a root event), the search terminates quickly.

### Return Value

`bool` — `True` means `source → target` (source causally precedes target). `False` means either target precedes source, or the events are concurrent. The caller must invoke `_reaches` in both directions to disambiguate.

### Algorithm

The function performs a **backward BFS** from `target` toward the roots of the event graph:

1. **Initialize** a visited set (using `id()` of event objects) and a queue seeded with `target`.
2. **Dequeue** the next event. If it **is** `source` (identity check, not equality), we've found a causal path — return `True`.
3. **Mark visited** using the object's `id()` to avoid revisiting the same event through different paths (a receive event has both a `_parent` and a `_cause`, so two paths can converge on the same ancestor).
4. **Enqueue predecessors**: the event's `_parent` (previous event on the same node) and `_cause` (the send event that triggered this receive, crossing a node boundary).
5. **Repeat** until the queue is empty. If exhausted without finding `source`, return `False`.

The key insight: the event graph has two kinds of edges — **process-local** edges (`_parent`: sequential events on one node) and **cross-node** edges (`_cause`: a send→receive link). Walking both edge types backward traces the full causal history, which is exactly Lamport's happens-before relation.

### Side Effects

None. Pure function — reads the event graph but never mutates it. The `visited` set is local.

### Error Handling

No explicit error handling. The function assumes:
- Events are valid `Event` objects (no `None` passed as `source` or `target`).
- `_parent` and `_cause`, if not `None`, are also valid `Event` objects.
- No cycles exist in the graph (an infinite loop would result otherwise, though this can't happen given how `Node` builds the graph).

### Usage Patterns

Called exclusively by `happens_before`, which wraps two calls:

```python
if _reaches(event_a, event_b):   # a → b?
    return True
if _reaches(event_b, event_a):   # b → a?
    return False
return None                       # concurrent
```

The caller is responsible for interpreting the three-valued result. `_reaches` itself is a private helper (underscore prefix) — not part of the public API.

### Dependencies

- `collections.deque` — for O(1) append/popleft BFS queue.
- `Event._parent` and `Event._cause` — the two edge types in the causal graph. These are set during event creation in `Node.local_event`, `Node.send_message`, and `Node.receive_message`.
- Python's `id()` builtin — used for visited-set membership since `Event` is a dataclass (default equality would compare fields, but we need object identity to distinguish events that might have identical timestamps and descriptions).

### Design Note

Using `id()` for the visited set rather than the events themselves is deliberate. `Event` is a `@dataclass`, which auto-generates `__eq__` based on field values. Two different event objects with the same `node_id`, `event_type`, `timestamp`, and `description` would compare equal under `__eq__`, but they're distinct nodes in the causal graph. The `id()` check ensures we track actual object identity, matching the `is` check used for the `source` comparison.

## Topics to Explore

- [function] `lamport-clocks/lamport.py:happens_before` — The public interface that calls `_reaches` twice to produce a three-valued causal ordering result
- [function] `lamport-clocks/lamport.py:Node.receive_message` — Where cross-node `_cause` edges are created, linking the receive event back to the send event
- [file] `lamport-clocks/test_lamport.py` — Test cases that exercise concurrent vs. causally-ordered event scenarios
- [general] `vector-clocks-vs-lamport` — Vector clocks can detect concurrency directly without BFS; understanding why Lamport clocks need graph traversal while vector clocks don't
- [function] `lamport-clocks/lamport.py:total_order` — How the total order (timestamp, node_id) relates to but differs from the partial causal order that `_reaches` computes

## Beliefs

- `reaches-uses-identity-not-equality` — `_reaches` uses `is` (object identity) to find the source event, not `==`, because dataclass equality would conflate distinct events with identical field values
- `reaches-walks-two-edge-types` — The causal graph has two edge types: `_parent` (same-node sequential) and `_cause` (cross-node send→receive), and `_reaches` must follow both to correctly determine happens-before
- `happens-before-requires-two-reaches-calls` — `happens_before` calls `_reaches` in both directions to distinguish "a before b", "b before a", and "concurrent" — a single call can only confirm or deny one direction
- `event-graph-acyclicity-by-construction` — `_reaches` assumes the event graph is a DAG with no cycle detection; this is safe because `Node` methods only ever link to previously-created events

