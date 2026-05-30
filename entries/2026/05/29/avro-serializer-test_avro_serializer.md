# File: avro-serializer/test_avro_serializer.py

**Date:** 2026-05-29
**Time:** 12:42

## Purpose

This is the test suite for the `avro_serializer` module — a from-scratch implementation of Apache Avro's binary serialization format. Its job is to verify that the implementation correctly handles Avro's encoding spec, schema evolution semantics, and compatibility checking. These are the core concepts from DDIA Chapter 4 (Encoding and Evolution), making this test file a specification-as-code for how schema-driven serialization should behave.

The tests are organized into 13 numbered groups that progress from low-level encoding primitives up to high-level schema registry operations, mirroring the layers of the Avro format itself.

## Key Components

**Test Groups (by number):**

| # | Focus | What it proves |
|---|-------|----------------|
| 1 | Primitive round-trips | Every Avro primitive type (`null`, `boolean`, `int`, `long`, `float`, `double`, `string`, `bytes`) survives encode→decode |
| 2 | Zigzag encoding | The variable-length integer encoding matches the Avro spec's canonical values |
| 3 | Complex types | Records, arrays, maps, unions, and enums all round-trip correctly |
| 4 | Schema evolution | The central DDIA concept — reading data written with an older/different schema |
| 5 | Compatibility checking | Static analysis of whether two schemas are compatible without encoding data |
| 6 | Schema registry | End-to-end: register schema, encode with ID, decode with a different reader schema |
| 7 | Nested schemas | Records containing arrays (composition) |
| 8 | Schema validation | Invalid schema definitions are rejected at construction time |
| 9 | Fixed type | Fixed-size binary blobs with size enforcement |
| 10 | Canonical equality | `Schema("int")` and `Schema({"type": "int"})` are treated as identical |
| 11 | Field reordering | Reader and writer can have the same fields in different order |
| 12 | Union disambiguation | When a union contains both a record and a map (both dicts in Python), the encoder picks the right branch |
| 13 | Int range validation | `int` enforces 32-bit signed bounds; `long` accepts larger values |

**Key imports from `avro_serializer`:**

- `Schema` — Parses and validates an Avro schema definition (string, dict, or list for unions)
- `AvroEncoder` / `AvroDecoder` — Stateless codec pair bound to a schema; decoder optionally takes both writer and reader schemas for evolution
- `SchemaRegistry` — Maps schema IDs to schemas, handles encode-with-ID / decode-with-ID
- `SchemaError` / `SchemaCompatibilityError` — Distinct exception types for invalid schemas vs. incompatible evolution
- `check_compatibility` — Static compatibility analysis returning a dict with `backward_compatible`, `full_compatible`, etc.
- `zigzag_encode` / `zigzag_decode` — The raw variable-length integer encoding functions

## Patterns

**Parametrized exhaustive coverage.** Tests 1 and 2 use `@pytest.mark.parametrize` to cover boundary values (0, -1, max int32, min int32, 2^40) and type variants in a single test function. This is the right pattern for codec testing where you want many inputs against the same logic.

**Two-schema decoder for evolution.** The `AvroDecoder(writer_schema, reader_schema)` constructor pattern appears in tests 4, 6, and 11. When only one schema is passed, the decoder reads with the same schema that wrote the data. When two are passed, it performs schema resolution — the core of Avro's evolution model.

**Assertion on binary representation.** `test_record` (test 3) asserts that field names (`b"id"`, `b"email"`) are absent from the encoded bytes. This validates a key Avro property: the binary format carries no field metadata, relying entirely on the schema for structure.

**Negative testing with `pytest.raises`.** Tests 4, 8, 9, and 13 verify that specific invalid operations produce the correct exception type, not just any error.

## Dependencies

**Imports:**
- `pytest` — test framework
- `avro_serializer` — the module under test (all public API surface)

**Imported by:** Nothing — this is a leaf test file. It is run by pytest and exists solely to validate `avro_serializer.py`.

## Flow

Each test follows the same pattern:

1. **Construct a `Schema`** from a type string or dict definition
2. **Create an `AvroEncoder`** bound to that schema
3. **Encode** a Python value → `bytes`
4. **Create an `AvroDecoder`** bound to the same or a different schema
5. **Decode** the bytes → Python value
6. **Assert** the decoded value matches the original (or the expected evolved form)

For schema evolution tests (4, 6, 11), step 4 uses a *different* reader schema, and step 6 asserts the value matches the reader's expectations (added defaults, reordered fields, promoted types).

## Invariants

- **Round-trip identity**: For any primitive or complex value, `decode(encode(x)) == x` when using the same schema.
- **Schema evolution contract**: When a reader schema adds a field with a default, the default fills in. When it adds a required field without a default, `SchemaCompatibilityError` is raised.
- **Type promotion is one-directional**: `int` → `long` and `int` → `double` work; the reverse is not tested (and presumably rejected).
- **Union disambiguation rule**: Record types match by field names; maps are the fallback for arbitrary string-keyed dicts.
- **Int32 bounds enforcement**: `int` type rejects values outside [-2^31, 2^31 - 1]; `long` accepts 2^40.
- **Fixed size enforcement**: A `fixed` type with `size: 4` rejects byte strings of any other length.
- **Schema canonical form**: String and dict representations of the same type are equal and produce the same hash.

## Error Handling

The test suite validates three error types:

| Exception | When raised | Test coverage |
|-----------|-------------|---------------|
| `SchemaError` | Invalid schema definition (unknown type, empty union, duplicate union branches, fixed missing name/size) | Tests 8, 9 |
| `SchemaCompatibilityError` | Reader schema has a required field that the writer didn't write and has no default | Test 4 |
| `ValueError` | Encoding a value that doesn't fit the schema's constraints (int overflow, wrong fixed size) | Tests 9, 13 |

Errors are tested with `pytest.raises` context managers — the tests verify the exception type but not the message content, which keeps them resilient to wording changes.

## Topics to Explore

- [file] `avro-serializer/avro_serializer.py` — The implementation behind all these tests; understanding the encoder/decoder internals (especially schema resolution in the two-schema decoder path)
- [function] `avro-serializer/avro_serializer.py:zigzag_encode` — The variable-length integer encoding that underpins all numeric types in Avro's binary format
- [function] `avro-serializer/avro_serializer.py:check_compatibility` — How the static compatibility checker determines backward/forward/full compatibility without encoding data
- [general] `avro-schema-evolution` — DDIA Chapter 4's treatment of how Avro handles schema changes differently from Protocol Buffers and Thrift (no field tags, relies on writer+reader schema pairing)
- [file] `avro-serializer/test_avro.py` — The other test file in this module; likely covers different aspects or was an earlier iteration

## Beliefs

- `avro-decoder-dual-schema` — `AvroDecoder` accepts one or two schemas; with two, it performs Avro schema resolution (writer schema for deserialization structure, reader schema for the output shape)
- `avro-binary-no-field-names` — Avro's binary encoding contains no field names or tags; the schema alone determines how bytes map to fields, verified by `test_record`'s byte-level assertion
- `avro-schema-canonical-equality` — `Schema("int")` and `Schema({"type": "int"})` are equal and hash-equal, enabling set/dict usage regardless of construction form
- `avro-union-disambiguation` — When a union contains both a record and a map type, the encoder resolves ambiguity by matching dict keys against record field names before falling back to map
- `avro-int-bounds-enforced` — The `int` type enforces 32-bit signed integer range at encode time via `ValueError`, while `long` accepts arbitrarily large values (at least 2^40)

