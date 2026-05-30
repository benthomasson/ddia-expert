# Topic: How Avro's block structure for arrays/maps enables skipping without parsing elements, and why the encoder never uses negative counts

**Date:** 2026-05-29
**Time:** 12:43

I don't have permission to read from the target repo at `/Users/ben/git/ddia-implementations/`. The observations only cover the first 200 lines of the 695-line file — enough to see the zigzag/varint primitives and the schema parser, but not the `AvroEncoder._encode` or `AvroDecoder._decode` methods where the array/map block encoding actually lives.

However, the entry at `entries/2026/05/29/avro-serializer-avro_serializer.md` was written from a full read of the file and documents the behavior in detail. Let me provide the explanation based on what we have, and flag the gap.

---

# Avro Block Encoding for Arrays and Maps

## How Block Encoding Works

Avro encodes arrays and maps using a **block structure**: a sequence of blocks, each prefixed with a count of how many elements it contains, terminated by a block with count `0`.

For an array like `[1, 2, 3]`, the wire format looks like:

```
[count: 3] [elem: 1] [elem: 2] [elem: 3] [count: 0]
```

Maps work identically, except each "element" is a key-value pair (string key followed by the value).

The count and the zero terminator are written via `write_long` / `read_long` (`avro_serializer.py:63-67`), which use zigzag + varint encoding. This means the count `3` encodes as zigzag `6`, which varints to a single byte. The terminating `0` is a single zero byte.

## How Negative Counts Enable Skipping

The Avro spec defines a dual meaning for the block count:

- **Positive count `N`**: "the next N elements follow immediately." The reader must parse each element to find the end of the block.
- **Negative count `-N`**: "the next N elements follow, AND the byte size of those N elements comes next as a long." The reader can either parse the elements or **jump ahead by the byte size** to skip the entire block without understanding the element schema.

This is critical for the `_skip` method described in the entry (`avro_serializer.py`, lines ~56 in the entry's description). When the decoder encounters a writer field whose type is an array or map that the reader doesn't care about, it needs to skip past that data. Without negative counts, skipping requires recursively descending into every element using the writer schema. With negative counts, the reader reads the byte-size and does a single `buf.seek(byte_size, 1)` — O(1) regardless of how many elements are in the block or how complex the element schema is.

## Why the Encoder Never Uses Negative Counts

As documented in the entry at `entries/2026/05/29/avro-serializer-avro_serializer.md:71`:

> Arrays and maps use Avro's block structure — a count followed by that many items, repeated until a 0-count block. Negative counts signal that a byte-size follows (for skip-ahead), though the encoder never produces negative counts.

The encoder always writes positive counts because:

1. **Computing byte-size requires double work.** To emit a negative count with a byte-size, the encoder would need to serialize all elements into a temporary buffer first (to measure the byte length), then write the count, then write the byte-size, then copy the serialized elements. This is a two-pass operation per block.

2. **The encoder doesn't know who will read the data.** Negative counts are an optimization for the *reader*, not the writer. The reader might parse every element anyway, making the byte-size overhead waste. The Avro spec makes negative counts optional for exactly this reason — producers can use the simpler positive-count encoding, and consumers that need skip capability handle it through other means (like parsing through with `_skip`).

3. **This implementation uses recursive `_skip` instead.** The decoder's `_skip` method (`avro_serializer.py`, described at line 56 of the entry) walks the writer schema to discard unwanted data without allocating Python objects. It's less efficient than a byte-size jump but doesn't require the encoder to pay the cost of computing block sizes.

This is a deliberate simplicity tradeoff in a teaching implementation. Production Avro writers (like the Java reference implementation) may emit negative counts to support readers that process large files and need to skip over array/map columns efficiently — particularly in Avro's object container file format where data is organized into large sync-marked blocks.

## The Decoder Must Handle Both

Even though this encoder only produces positive counts, the decoder reads via `read_long` which returns a zigzag-decoded signed integer. A robust decoder should handle negative counts from data produced by other Avro encoders. The entry confirms the decoder is structured around this duality — it reads a count, and if non-zero, processes that many elements before reading the next count.

## Topics to Explore

- [function] `avro-serializer/avro_serializer.py:_skip` — The recursive skip implementation that compensates for not having byte-size blocks; trace how it handles nested arrays and records without allocating
- [function] `avro-serializer/avro_serializer.py:_encode` — Read the actual array/map encoding branches (lines ~200-400) to confirm block structure; the observations only cover the first 200 lines
- [function] `avro-serializer/avro_serializer.py:_resolve_record` — See how _skip is called for writer fields the reader doesn't have — this is where block-skip would matter most in practice
- [general] `avro-object-container-format` — Avro's file format adds sync markers and codec headers around blocks of records; this is where negative counts and skip-ahead become essential for columnar-style access patterns

## Beliefs

- `avro-block-encoding-zero-terminated` — Arrays and maps are encoded as a sequence of count-prefixed blocks terminated by a 0-count block; counts and elements use zigzag+varint via `write_long`/`read_long`
- `avro-negative-count-carries-byte-size` — A negative block count `-N` in Avro's spec means N elements follow, preceded by a byte-size long that enables O(1) skipping of the entire block without parsing elements
- `avro-encoder-positive-counts-only` — This implementation's `AvroEncoder` always writes positive counts for array/map blocks, never negative counts with byte-size prefixes, trading skip efficiency for encoder simplicity
- `avro-skip-is-recursive-not-bytejump` — The decoder's `_skip` method discards unwanted fields by recursively walking the writer schema rather than using byte-size jumps, which works correctly but requires understanding the element schema

**Note:** The observations only cover the first 200 lines of the 695-line `avro_serializer.py`. The actual `_encode` and `_decode` methods for arrays and maps were not directly visible. The explanation above is based on the detailed entry at `entries/2026/05/29/avro-serializer-avro_serializer.md` (which was written from a full read) and knowledge of the Avro spec. To confirm the exact implementation, read lines 200+ of `avro-serializer/avro_serializer.py`.

