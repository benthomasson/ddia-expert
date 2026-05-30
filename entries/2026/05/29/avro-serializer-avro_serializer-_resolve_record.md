# Function: _resolve_record in avro-serializer/avro_serializer.py

**Date:** 2026-05-29
**Time:** 12:44



# `_resolve_record` — Avro Schema Evolution for Records

## Purpose

`_resolve_record` implements **schema resolution** between two record schemas — a writer schema (used when the data was serialized) and a reader schema (what the current application expects). This is the core mechanism enabling Avro's schema evolution: it lets you read data written with an older (or newer) schema using a different schema, as long as they're compatible.

This exists because in real systems, schemas change over time. A producer might add a field, remove one, or reorder them. Without resolution, every consumer would need to use the exact schema the data was written with. `_resolve_record` bridges that gap.

## Contract

**Preconditions:**
- `buf` is a readable `io.BytesIO` positioned at the start of a serialized record
- `writer` and `reader` are both `Schema` objects with `type_name == "record"` and matching `name` properties (the caller in `_decode` enforces the name check before calling)
- The bytes in `buf` are laid out in **writer field order** — this is how Avro binary encoding works (no field tags, just sequential values)

**Postconditions:**
- `buf` is advanced past all writer fields (consumed or skipped)
- Returns a `dict` with exactly the reader's fields, in reader field order
- Every reader field has a value — either decoded from the wire, or filled from its default

**Invariants:**
- Writer fields absent from the reader are fully consumed (skipped) from the buffer — never left dangling
- Reader fields absent from the writer must have a default, or resolution fails

## Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `buf` | `io.BytesIO` | Binary stream positioned at the start of the record's serialized data. Must contain enough bytes for all writer fields. |
| `writer` | `Schema` | The schema used when the data was encoded. Determines what's on the wire and in what order. |
| `reader` | `Schema` | The schema the caller wants the data shaped as. Determines which fields appear in the result and their order. |

**Edge cases:** If writer and reader have identical fields, this degenerates to a simple sequential decode. If the writer has zero fields, the result is built entirely from reader defaults.

## Return Value

A `dict` mapping field names (strings) to decoded values. The keys are exactly the reader's field names. The caller (`_decode`) returns this directly — no further transformation is applied.

**Caller must handle:** The returned dict always has all reader fields. No `None` sentinels for missing fields — if a field can't be resolved, the method raises rather than returning a partial result.

## Algorithm

The method works in two passes:

### Pass 1 — Consume the wire data (writer field order)

```
for each field in writer.fields (in serialized order):
    if field exists in reader:
        decode it using both writer and reader field types (enables type promotion)
        store in writer_values dict
    else:
        skip the bytes (read and discard) — the reader doesn't want this field
```

This pass **must** iterate in writer field order because Avro binary format has no field delimiters or tags. Each field's size is determined by its schema, so you must read (or skip) every field sequentially to know where the next one starts.

### Pass 2 — Build the result (reader field order)

```
for each field in reader.fields:
    if field was decoded from writer → use it
    elif field has a default → use the default
    else → raise SchemaCompatibilityError
```

The two-pass design decouples wire order (writer) from output order (reader). Note that `writer_field_map` is constructed but never read — it's dead code, likely left from a refactor.

## Side Effects

- **Advances `buf`** past all writer fields. This is the primary side effect and is essential — the buffer must be correctly positioned for whatever comes next in the stream.
- Recursively calls `self._decode()` for field values, which may itself recurse into nested records, arrays, maps, etc.
- Calls `self._skip()` for discarded fields, which also advances `buf`.

No I/O, no state mutations on `self`, no logging.

## Error Handling

| Condition | Exception | Message |
|-----------|-----------|---------|
| Reader field missing from writer, no default | `SchemaCompatibilityError` | `"Reader field '{name}' has no default and is missing from writer"` |
| Type incompatibility in a shared field | `SchemaCompatibilityError` | Raised by recursive `_decode` (e.g., can't resolve `int` to `string`) |
| Truncated buffer | `ValueError` | Raised by `read_varint` or `struct.unpack` deeper in the call stack |

No errors are swallowed. Every failure path either raises or propagates an exception from a callee.

## Usage Patterns

Called exclusively from `_decode` when both writer and reader are `"record"` type with matching names:

```python
# In _decode:
if wt == "record" and rt == "record":
    if writer.name != reader.name:
        raise SchemaCompatibilityError(...)
    return self._resolve_record(buf, writer, reader)
```

Typical end-user scenario — decoding data written with schema v1 using schema v2:

```python
v1 = Schema({"type": "record", "name": "User", "fields": [
    {"name": "name", "type": "string"},
    {"name": "age", "type": "int"}
]})
v2 = Schema({"type": "record", "name": "User", "fields": [
    {"name": "name", "type": "string"},
    {"name": "age", "type": "int"},
    {"name": "email", "type": "string", "default": ""}
]})
decoder = AvroDecoder(writer_schema=v1, reader_schema=v2)
result = decoder.decode(data)  # result["email"] == ""
```

## Dependencies

- `self._decode()` — recursive descent for field values; handles type promotion via `PROMOTIONS`
- `self._skip()` — consumes bytes for unwanted fields without materializing values
- `Schema.fields` — provides field lists with `name`, `type`, and optional `default`
- No external library dependencies — pure Python with `struct` and `io` from stdlib

## Unforced Assumptions

1. **`writer_field_map` is constructed but unused** — a minor dead-code issue, not a bug.
2. **Defaults are used as-is (no deep copy)** — if a default is a mutable value (list/dict), multiple records could share the same default object. Mutating one would affect others.
3. **No cycle detection for nested records** — a record containing itself would recurse infinitely. The `Schema` parser doesn't support recursive types, so this can't happen in practice, but it's not enforced at this level.
4. **Field order in the result dict relies on Python 3.7+ insertion-order guarantee** — the code iterates reader fields sequentially and inserts into a plain `dict`.

---

## Topics to Explore

- [function] `avro-serializer/avro_serializer.py:_skip` — The counterpart that consumes wire bytes without decoding; essential for understanding how removed fields are handled
- [function] `avro-serializer/avro_serializer.py:_decode` — The dispatch layer that routes to `_resolve_record`; shows how type promotion and union resolution compose with record resolution
- [function] `avro-serializer/avro_serializer.py:_resolve_check` — The dry-run compatibility checker that mirrors this logic without actually reading bytes
- [file] `avro-serializer/test_avro_serializer.py` — Test cases that exercise schema evolution scenarios (added fields, removed fields, type promotion within records)
- [general] `avro-schema-resolution-spec` — The Avro specification's schema resolution rules that this code implements; compare to see if edge cases like aliased field names are handled

## Beliefs

- `resolve-record-two-pass` — `_resolve_record` uses a two-pass algorithm: first pass consumes all writer fields from the buffer in wire order, second pass assembles the result in reader field order
- `resolve-record-default-fill` — Reader fields missing from the writer are filled with their declared default value; if no default exists, `SchemaCompatibilityError` is raised
- `resolve-record-skip-unknown` — Writer fields not present in the reader schema are fully consumed from the buffer via `_skip` and discarded, ensuring the stream stays correctly positioned
- `resolve-record-dead-code` — `writer_field_map` is constructed but never referenced in `_resolve_record`, making it dead code
- `resolve-record-shared-defaults` — Default values are inserted by reference without copying, so mutable defaults (lists, dicts) would be shared across decoded records

