# File: avro-serializer/test_avro.py

**Date:** 2026-05-29
**Time:** 12:46

## Purpose

`test_avro.py` is the primary integration test suite for the Avro serializer module. It validates that the `avro_serializer.py` implementation correctly handles the full Avro specification surface: primitive types, complex types (records, arrays, maps, unions, enums), schema evolution with forward/backward compatibility, type promotion, a schema registry, and streaming decode. It's a standalone script (not pytest-based) — run via `python test_avro.py` — and doubles as a specification-by-example document for how the serializer API works.

## Key Components

**Primitive round-trip** (`test_primitives`): Exercises every Avro primitive type — `null`, `boolean`, `int`, `long`, `float`, `double`, `string`, `bytes` — through encode/decode, including boundary values (`2147483647`, `-2147483648`, `2**40`). Float comparison uses an epsilon tolerance (`1e-6`) since IEEE 754 floats lose precision in round-trips.

**Zigzag encoding** (`test_zigzag`): Validates the varint zigzag encoding (mapping signed integers to unsigned for compact representation). Tests the critical values: 0, ±1, ±2, and the int32 boundaries.

**Complex types** (`test_record`, `test_array`, `test_map`, `test_union`, `test_enum`): Each validates encode/decode round-trips. `test_record` also asserts that field names are absent from the binary output (`b"email" not in data`), verifying Avro's schemaless wire format.

**Schema evolution** (`test_schema_evolution_add_field`, `test_type_promotion`, `test_enum_evolution`): The core of what makes Avro interesting. Tests that:
- A reader with a new field (with default) can decode data written with an older schema — backward compatibility.
- A reader can silently drop fields present in the writer schema but absent in the reader schema (the `email` field disappears).
- Primitive types promote correctly: `int→long`, `int→float`, `int→double`, `long→double`, `float→double`.
- Enum evolution falls back to a declared `default` symbol when the writer sends an unknown symbol.

**Incompatibility** (`test_incompatibility`): Negative test — adding a required field (no default) to the reader schema must raise `SchemaCompatibilityError`.

**Compatibility check** (`test_compatibility_check`): Tests the standalone `check_compatibility()` function that reports backward/forward/full compatibility between two schemas without actually encoding data.

**Schema registry** (`test_schema_registry`): Tests `SchemaRegistry` — register a schema, encode with a schema ID prefix, decode with a different reader schema. This mirrors the Confluent Schema Registry pattern where the wire format is `[schema_id][avro_payload]`.

**Nested structures** (`test_nested`): Records containing arrays and maps, and records nested inside records.

**Streaming** (`test_streaming`): Concatenates two encoded integers into a single byte buffer, then decodes them sequentially from a `BytesIO` stream, verifying exact byte consumption (no leftover bytes).

**Performance** (`test_performance`): Benchmarks 10,000 record encode/decode cycles. No assertion on timing — purely informational output.

## Patterns

**Manual test harness**: Uses `print()` + `assert` instead of pytest. Each test function prints its category name and sub-results, ending with `ALL TESTS PASSED`. This is common in DDIA reference implementations — self-contained, no external test framework dependency.

**Round-trip testing idiom**: Almost every test follows `encode(value) → decode(bytes) → assert == value`. This validates the encode/decode pair as inverses.

**Writer/reader schema separation**: `AvroDecoder` takes an optional second schema argument — `AvroDecoder(writer_schema, reader_schema)`. When omitted, writer and reader are the same. This two-schema design is the key Avro abstraction for schema evolution.

**Negative testing via try/except**: `test_incompatibility` uses a try/except/assert-False pattern instead of `pytest.raises`. Idiomatic for framework-free test files.

## Dependencies

**Imports from** `avro_serializer`:
- `Schema` — schema definition object
- `AvroEncoder` / `AvroDecoder` — encode/decode engines
- `SchemaRegistry` — schema ID management
- `SchemaError`, `SchemaCompatibilityError` — error types
- `check_compatibility` — standalone compatibility checker
- `zigzag_encode`, `zigzag_decode` — low-level varint primitives

**Standard library**: `io.BytesIO` (streaming test), `time` (performance benchmark).

**Nothing imports this file** — it's a leaf test module.

## Flow

Execution is linear when run as `__main__`: zigzag → primitives → record → array → map → union → enum → evolution → promotion → incompatibility → compatibility check → registry → nested → compact → streaming → enum evolution → performance. Each test is independent — no shared state between tests.

The `AvroDecoder` call with two schemas (`AvroDecoder(v1, v2).decode(data)`) is the most important flow to understand: it reads bytes according to `v1`'s field layout, then resolves fields against `v2`'s expectations — filling defaults for missing fields, dropping fields not in `v2`, and promoting types where allowed.

## Invariants

1. **Round-trip fidelity**: For same-schema encode/decode, output must equal input (exact for all types except float, which uses epsilon).
2. **No field names on the wire**: Record binary output must not contain field name strings — Avro encodes by field order, not by name.
3. **Backward compatibility requires defaults**: A reader schema can add fields only if they have defaults. Without a default, `SchemaCompatibilityError` is raised.
4. **Exact byte consumption**: Streaming decode must consume exactly the bytes for each value, leaving no residual bytes in the stream.
5. **Type promotion is directional**: Only widening promotions are valid (`int→long`, never `long→int`).

## Error Handling

Only one error path is explicitly tested: `SchemaCompatibilityError` in `test_incompatibility`. The test uses a try/except pattern and `assert False` as the failure branch. `SchemaError` is imported but not directly tested in this file (likely tested in `test_avro_serializer.py`). No errors are swallowed — all assertions fail loudly.

## Topics to Explore

- [file] `avro-serializer/avro_serializer.py` — The implementation behind all these tests; see how `_decode` handles writer/reader schema resolution and type promotion
- [function] `avro-serializer/avro_serializer.py:zigzag_encode` — The varint encoding that makes Avro integers compact; key to understanding why small numbers take fewer bytes
- [file] `avro-serializer/test_avro_serializer.py` — The companion test file; likely covers edge cases, error paths, and `SchemaError` scenarios not tested here
- [general] `avro-schema-resolution` — Avro spec's schema resolution rules (Chapter 4 of DDIA) — how reader/writer schema mismatches are reconciled at decode time
- [general] `confluent-schema-registry-wire-format` — The `[magic_byte][schema_id][payload]` framing that `SchemaRegistry.encode_with_id` implements

## Beliefs

- `avro-no-field-names-on-wire` — Avro record binary encoding contains no field names; fields are identified purely by schema-defined order, verified by `test_record` and `test_compact_encoding`
- `avro-backward-compat-requires-defaults` — A reader schema that adds a field without a default value causes `SchemaCompatibilityError` during decode; backward compatibility demands defaults on all new fields
- `avro-decoder-two-schema-resolution` — `AvroDecoder` accepts `(writer_schema, reader_schema)` and resolves fields between them at decode time — filling defaults for added fields and silently dropping removed fields
- `avro-type-promotion-widening-only` — Type promotion supports `int→long`, `int→float`, `int→double`, `long→double`, `float→double` — strictly widening conversions, no narrowing
- `avro-enum-default-fallback` — When a writer sends an enum symbol not in the reader's symbol list, the reader falls back to the `default` symbol declared in the reader schema

