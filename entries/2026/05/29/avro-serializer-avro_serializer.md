# File: avro-serializer/avro_serializer.py

**Date:** 2026-05-29
**Time:** 08:52

# `avro-serializer/avro_serializer.py`

## Purpose

This file is a from-scratch implementation of Apache Avro's binary serialization format, built to demonstrate the schema evolution concepts from DDIA Chapter 4 (Encoding and Evolution). It owns the entire encode/decode pipeline: schema parsing, binary serialization, schema resolution (reading data written with one schema using a different schema), compatibility checking, and a simple schema registry. This is not a wrapper around the `avro` library — it reimplements the wire format and resolution rules directly.

## Key Components

### Constants and Configuration

- **`PRIMITIVES`** — the eight Avro primitive type names. Used as the ground truth for type validation throughout.
- **`_NO_DEFAULT`** — sentinel object for distinguishing "no default provided" from `None` as a default value. Used in canonical form computation.
- **`PROMOTIONS`** — set of `(from, to)` pairs defining which type widening is legal during schema resolution: `int→long→float→double`, plus `int→float`, `int→double`, `long→double`. This is the Avro spec's promotion rule.

### Zigzag / Varint Encoding (`zigzag_encode`, `zigzag_decode`, `write_varint`, `read_varint`, `write_long`, `read_long`)

Avro encodes integers using zigzag encoding on top of variable-length encoding. Zigzag maps signed integers to unsigned (`0→0, -1→1, 1→2, -2→3, ...`) so small-magnitude negatives use few bytes. `write_long`/`read_long` compose zigzag with varint and are the workhorses — they encode every integer in the format: lengths, array counts, union indices, enum ordinals, and the `int`/`long` types themselves.

### `Schema`

Parses an Avro-like schema definition (JSON-compatible Python dicts/lists/strings) into an internal representation. Validates structure eagerly in `_parse`:

- **Strings** must be known primitives.
- **Lists** become unions (no duplicate types allowed, at least one branch required).
- **Dicts** are dispatched on `"type"` — supports `record`, `array`, `map`, `enum`, `fixed`, and bare primitives.

The class exposes typed properties (`fields`, `items`, `values`, `symbols`, `union_types`, `size`) that return empty/None for inapplicable types. `_canonical_form()` produces a hashable tuple representation for equality/hashing — two `Schema` objects with structurally identical definitions are equal regardless of construction path.

### `AvroEncoder`

Serializes Python values to bytes given a writer schema. The `_encode` method dispatches on `schema.type_name`:

- Primitives: `null` writes nothing, `boolean` writes one byte, `int`/`long` use zigzag+varint, `float`/`double` use little-endian IEEE 754.
- `string`/`bytes`: length-prefixed with a zigzag long.
- `record`: fields encoded in declaration order — no field names in the output.
- `array`/`map`: block-encoded with a count prefix, terminated by a 0-count block.
- `union`: writes the branch index as a long, then the value.
- `enum`: writes the symbol's ordinal index.
- `fixed`: writes raw bytes (must match declared size exactly).

`_match_union` resolves which union branch to use via Python type inspection. It checks `bool` before `int` (since `bool` is an `int` subclass in Python), and for ambiguous dict values, prefers `record` if the keys match field names.

### `AvroDecoder`

Reads binary data using a **writer schema** (what the data was written with) and resolves it against a **reader schema** (what the consumer expects). This is the core of schema evolution. The `_decode` method handles:

- **Union resolution**: reads the branch index from the wire, then finds the matching branch in the reader's union (or the reader itself if it's not a union).
- **Primitive promotion**: `int→long`, `int→float`, etc., via `_promote`.
- **Record resolution** (`_resolve_record`): reads all writer fields in wire order, matches them to reader fields by name, skips unknown writer fields, and fills missing reader fields from defaults. This is where forward/backward compatibility happens.
- **Enum resolution**: maps by symbol name, falls back to enum-level default for unknown symbols.
- **`_skip`**: reads and discards a value without allocating — used when the writer has a field the reader doesn't care about.

### `check_compatibility` / `_resolve_check`

Performs a dry-run of schema resolution without any data, checking whether a reader can consume writer-produced data (backward compatibility) and vice versa (forward compatibility). Returns a dict with `backward_compatible`, `forward_compatible`, `full_compatible`, and a list of `errors`.

### `SchemaRegistry`

A minimal in-memory registry that assigns auto-incrementing integer IDs to schemas. `encode_with_id` prepends a 4-byte big-endian schema ID to the encoded payload; `decode_with_id` strips it and looks up the writer schema. This models Confluent's Schema Registry pattern where the writer schema ID travels with the message.

## Patterns

- **Recursive descent**: Both `Schema._parse`, `AvroEncoder._encode`, and `AvroDecoder._decode` use recursive dispatch on the type discriminator. The schema tree and the encode/decode logic mirror each other structurally.
- **Writer-reader resolution**: The decoder always takes two schemas. This is Avro's defining characteristic vs. Protobuf/Thrift — the wire format contains no field tags or type markers, so you need the writer schema to parse the bytes and the reader schema to know what the consumer expects.
- **Sentinel pattern**: `_NO_DEFAULT` avoids conflating "default is None" with "no default specified" — important because `null` is a valid Avro default.
- **Block encoding for collections**: Arrays and maps use Avro's block structure — a count followed by that many items, repeated until a 0-count block. Negative counts signal that a byte-size follows (for skip-ahead), though the encoder never produces negative counts.

## Dependencies

**Imports**: Only stdlib — `io.BytesIO` for buffer management, `struct` for IEEE 754 float encoding and the 4-byte schema ID header. No external dependencies.

**Imported by**: `test_avro.py` and `test_avro_serializer.py` — two test suites that exercise the serializer and schema evolution scenarios.

## Flow

**Encode path**: caller creates `Schema(definition)` → creates `AvroEncoder(schema)` → calls `encoder.encode(value)` → recursive `_encode` walks the schema tree and writes bytes to a `BytesIO` buffer → returns `bytes`.

**Decode path**: caller creates `AvroDecoder(writer_schema, reader_schema)` → calls `decoder.decode(data)` → recursive `_decode` reads from a `BytesIO` buffer using the writer schema to know the wire layout, but returns values shaped by the reader schema.

**Registry path**: `registry.encode_with_id(sid, value)` → encodes normally → prepends 4-byte ID. `registry.decode_with_id(data, reader_schema)` → strips ID → looks up writer schema → decodes with resolution.

## Invariants

1. **Field order is wire order**: Record fields are encoded/decoded in the order declared in the **writer** schema. Reordering fields in the schema changes the wire format.
2. **No self-describing format**: The binary output contains no type tags, field names, or schema metadata. You cannot decode without the writer schema.
3. **Names must match for named types**: Records, enums, and fixed types require matching `name` fields during resolution — you can't rename a type and maintain compatibility.
4. **Promotion is one-directional**: `int→long` works, `long→int` does not. The `PROMOTIONS` set defines all legal promotions.
5. **Missing reader fields require defaults**: During record resolution, if the reader has a field not present in the writer, it must have a `default` or resolution fails with `SchemaCompatibilityError`.
6. **Union branch uniqueness**: A union cannot contain two branches with the same type name (or record name), enforced at parse time.
7. **Schema IDs are 4-byte big-endian unsigned**: The registry prepends `struct.pack('>I', schema_id)` — schema IDs are limited to `[0, 2^32)`.

## Error Handling

Two custom exceptions partition the error space:

- **`SchemaError`** — raised during `Schema.__init__` for structurally invalid definitions (unknown types, missing names, duplicate fields). This is a parse-time error; if you have a `Schema` object, it's valid.
- **`SchemaCompatibilityError`** — raised during decode resolution or compatibility checking when schemas can't be reconciled (name mismatches, missing fields without defaults, incompatible types).
- **`ValueError`** — raised during encoding for data-level issues (value out of range for `int`, wrong byte count for `fixed`, no matching union branch) and during decoding for truncated input.
- **`KeyError`** — raised by `SchemaRegistry.get` for unknown schema IDs.

Errors are never swallowed. The compatibility checker catches `SchemaCompatibilityError` internally to report rather than raise, but the check functions always return a result.

## Topics to Explore

- [file] `avro-serializer/test_avro_serializer.py` — See what schema evolution scenarios are tested: field addition/removal, type promotion, enum symbol changes
- [general] `avro-block-encoding` — How Avro's block structure for arrays/maps enables skipping without parsing elements, and why the encoder never uses negative counts
- [function] `avro-serializer/avro_serializer.py:_resolve_record` — The heart of schema evolution — trace through a scenario where writer has fields A,B,C and reader has B,C,D with default
- [general] `confluent-schema-registry-protocol` — Compare `SchemaRegistry.encode_with_id` to Confluent's wire format (magic byte + 4-byte ID) to see what's simplified here
- [file] `avro-serializer/test_avro.py` — Likely tests the lower-level encoding/decoding without schema evolution, good for understanding the wire format

## Beliefs

- `avro-no-self-describing` — The binary encoding contains no type information; decoding requires the exact writer schema that produced the bytes
- `avro-record-field-order-is-wire-order` — Record fields are serialized in schema declaration order with no field identifiers, so reordering fields in the writer schema is a breaking change
- `avro-promotion-asymmetric` — Type promotion during resolution is one-directional (int→long, int→float, int→double, long→float, long→double, float→double); reverse promotion is a `SchemaCompatibilityError`
- `avro-schema-equality-structural` — Two `Schema` instances are equal if their canonical forms match, regardless of how they were constructed or whether they carry different defaults
- `avro-registry-id-prefix` — `SchemaRegistry` prepends a 4-byte big-endian unsigned int schema ID to encoded payloads, with no magic byte (unlike Confluent's 5-byte header)

