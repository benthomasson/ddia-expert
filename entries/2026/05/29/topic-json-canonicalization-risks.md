# Topic: Whether `json.dumps(sort_keys=True)` is truly canonical (it isn't for floats, nested dicts with mixed types, or non-ASCII) and when this matters

**Date:** 2026-05-29
**Time:** 11:52

# Is `json.dumps(sort_keys=True)` Truly Canonical?

**Short answer: no, but it's good enough for this codebase — barely.**

The only place `sort_keys=True` is used for hashing is `byzantine-fault-tolerance/pbft.py:46`:

```python
return hashlib.sha256(json.dumps(request, sort_keys=True, default=str).encode()).hexdigest()
```

This creates a digest that all PBFT replicas must agree on. If two honest nodes compute different hashes for the same logical request, consensus breaks. That makes this line load-bearing for correctness.

## Where `sort_keys=True` Falls Short

**Floats are not round-trip stable.** `json.dumps(0.1 + 0.2)` produces `0.30000000000000004`. The exact string representation of a float can vary across Python versions and platforms. If a request payload contains floating-point arithmetic results, two nodes running different Python builds could hash the same logical value differently.

**`default=str` is a canonicalization escape hatch.** Any non-JSON-serializable object gets converted via `str()`. This is dangerous — `str(datetime.now())` includes microseconds, `str(some_object)` may include memory addresses, and `str()` output for custom classes is whatever `__repr__` happens to return. Two nodes receiving the same request object could serialize it differently if any field falls through to `default=str`.

**Non-ASCII strings have multiple valid JSON encodings.** By default Python's `json.dumps` uses `ensure_ascii=True` (escaping to `\uXXXX`), which is consistent within CPython. But if that default ever changes or someone passes `ensure_ascii=False`, the same Unicode string can be encoded as either the literal character or the escape sequence — both valid JSON, different bytes, different hashes.

**Key ordering is only skin-deep.** `sort_keys=True` sorts top-level and nested dict keys, but it sorts them lexicographically as Python strings. This is fine for ASCII keys but can produce different orderings depending on locale-aware string comparison in edge cases.

## Why It Works Here Anyway

This is a reference implementation. The PBFT `request` dicts are constructed internally with simple string keys and string/integer values — no floats, no custom objects, no Unicode surprises. The `default=str` is a safety net that probably never fires in normal operation. In this controlled context, `json.dumps(sort_keys=True)` produces consistent output.

## Contrast with the Rest of the Codebase

Every other `json.dumps` call in the codebase is for **storage**, not **hashing**:

| File | Line | Purpose |
|------|------|---------|
| `event-sourcing-store/event_store.py` | 132 | Append to WAL |
| `batch-word-count/pipeline.py` | 134, 147, 242, 320 | Write intermediate records |
| `partitioned-log/partitioned_log.py` | 427 | Write to partition files |
| `unbundled-database/unbundled_database.py` | 50 | Persist log entries |

None of these use `sort_keys=True` because they don't need to — they're writing data to be read back by `json.loads`, not compared byte-for-byte. Serialization roundtripping doesn't require canonical form.

The Merkle tree (`merkle-tree/merkle_tree.py:12`) avoids the problem entirely by hashing raw `bytes` directly:

```python
def _sha256(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()
```

Callers provide the bytes — no serialization ambiguity. This is the correct approach when hash stability is critical.

## When Would This Actually Break?

If someone extended the PBFT implementation to handle requests with:
- Floating-point values (prices, coordinates, timestamps-as-floats)
- Nested objects with `datetime` or custom types (hitting `default=str`)
- User-supplied Unicode content in keys or values

Any of these would risk hash divergence across replicas. A production system would use a proper canonical serialization format (e.g., [RFC 8785 JSON Canonicalization Scheme](https://datatracker.ietf.org/doc/html/rfc8785), canonical CBOR, or protobuf).

---

## Topics to Explore

- [function] `byzantine-fault-tolerance/pbft.py:_digest` — Trace all callers to see what `request` dicts actually contain and whether non-canonical inputs can reach this function
- [file] `merkle-tree/merkle_tree.py` — Compare its bytes-first hashing approach with PBFT's JSON-then-hash approach as two different canonicalization strategies
- [general] `rfc-8785-json-canonicalization` — The IETF standard for deterministic JSON serialization, which solves exactly the problems `sort_keys=True` doesn't
- [function] `event-sourcing-store/event_store.py:_persist_event` — Examine whether the WAL's non-canonical JSON could cause issues during log replay if the writing and reading Python versions differ
- [general] `protobuf-vs-json-for-consensus` — Why production BFT systems (e.g., Tendermint) use binary serialization formats instead of JSON for message digests

## Beliefs

- `pbft-sole-sort-keys-user` — `byzantine-fault-tolerance/pbft.py:46` is the only call site that uses `json.dumps(sort_keys=True)` for hashing; all other `json.dumps` calls are for storage serialization where canonical form is irrelevant
- `pbft-default-str-fragile` — The `default=str` parameter in the PBFT digest function silently converts non-serializable objects to their `str()` representation, which is not deterministic across replicas for types like `datetime` or custom objects
- `merkle-tree-hashes-raw-bytes` — The Merkle tree implementation avoids serialization canonicalization entirely by hashing caller-provided `bytes` directly via `hashlib.sha256(data)`, making it immune to JSON encoding ambiguity
- `json-dumps-sort-keys-insufficient-for-floats` — `json.dumps(sort_keys=True)` does not guarantee canonical output for floating-point values, as Python's float-to-string conversion can vary across versions and platforms

