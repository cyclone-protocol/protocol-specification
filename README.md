# Cyclone Protocol

**English** · [Vietnamese](vi/README.md)

Cyclone defines how data is written as bytes.

That’s it.

---

## A 30-second explanation

You have a `Player` object:

```
Hp   = 100
Name = "Alice"
```

Cyclone specifies that it becomes exactly this byte sequence:

```
64 00 00 00   05 00 00 00   41 6C 69 63 65
Hp = 100    Length = 5      A  l  i  c  e
```

13 bytes. No field names, no brackets, no tags - just raw data.

The key point: Go, C#, Rust, and C must all produce these exact 13 bytes. Not just "equivalent," but byte-for-byte identical.

---

## Why is this needed?

With JSON or Protobuf, the same data can produce different byte sequences depending on the language, library, or version. Usually, that doesn't matter. But it causes issues when:

- You sign or hash the data (payload) - signatures are based on bytes; if the bytes change, the signature becomes invalid even if the data remains the same.
- You run replay simulations or lockstep synchronization - two computers must produce the same result from the same input sequence.
- You want to verify SDK compatibility - comparing bytes is the only way.

Cyclone solves this by eliminating choices: whenever an encoder has a choice to make, different encoders will inevitably choose differently.

---

## What Cyclone is not

```
- Networking framework    - RPC
- Game engine             - Transport (TCP/UDP/QUIC)
- Encryption              - Compression
```

Those things are built on top of Cyclone. Cyclone operates strictly on data format conversions:

```
Model  →  Bytes
Bytes  →  Model
```

---

## How it works

Understanding just four rules is sufficient to grasp the entire system:

1. Numbers are fixed-size and little-endian. A `UInt32` always occupies 4 bytes, even if the value is 0. Varints (variable-size integers) are not used.

2. Strings and arrays are prefixed with their length: `[4-byte length][data]`. For strings, the length represents the number of UTF-8 bytes, not the character count.

3. Models consist of contiguous fields. There are no headers, padding, or delimiters. The decoder identifies the current field based on the cursor position, not via tags.

4. There is no metadata. The byte stream is not self-describing; the receiver must already know the exact order and data type of each field.

Consequences of rules 3 and 4 - important to understand before use:

```
Changing field names      →  no change to bytes            (safe)
Changing field order      →  changes the entire byte stream (breaks compatibility)
Incorrect definition      →  usually no error; produces incorrect data without warning
```

---

## Cyclone does not require a specific IDL

The source of truth is the **Schema** - a set comprising data type names, field names, Cyclone types, and field order. The byte stream is generated from the Schema, and *only* from that Schema. However, Cyclone does not dictate *how* you must write that Schema:

```
Multiple representations   →   one Schema   →   exactly one byte sequence
```

There is no requirement for a `.cyclone` file, nor is a specific IDL compiler mandatory. As long as both parties generate the same Schema, the resulting byte output will match perfectly.

The reference implementation chooses to represent the schema using annotations directly on existing models - for example, in C#:

```csharp
[Network]
public partial class Player
{
    [Network(UInt32)]
    public uint Hp;

    [Network(PlayerInfo)]
    public PlayerInfo Info;
}
```

It does not need to understand keywords like `public`, `partial`, `class`, or `uint`, nor whether `PlayerInfo` is a class or a struct. It simply extracts three specific elements to construct the Schema:

```
Type Name        →  Player
Field Name       →  Hp
Cyclone Type     →  UInt32
```

The Cyclone type is determined by the annotation rather than being inferred from the host language's type system. Applying `[Network(UInt32)]` to an incompatible field results in a declaration error - Cyclone does not make assumptions on your behalf.

Consequently, adding support for a new language becomes a simple, lightweight task. The frontend component only needs to read annotations, type names, and field names; it requires no semantic analysis or deep understanding of the target language's type system.

A different implementation could easily adopt an alternative approach - such as using separate schema files, macros, or manually writing codec code. There is only one standard for evaluation: whether the generated data bytes are correct.
---
## When NOT to use it

It is important to clarify that:

- Schemas change frequently → modifications must avoid altering field order or removing fields from the middle; fields may only be added or removed at the end.
- There are many "optional" fields → Version v1 lacks support for Optional/Nullable types.

If any of the above apply to your use case, Protobuf would be a better choice.

---

## Cyclone's Guarantee

```
Same schema, same values

↓

All implementations MUST produce identical byte sequences.
```

Any discrepancy indicates an implementation error, not a protocol error. Such errors can be detected by comparing byte arrays - please refer to the Conformance documentation.

---

## Documentation

Please read in the following order:

| Document | Answers the question |
|----------|-----------------|
| [RFC-0001 - What is Cyclone](RFC-0001.md) | What problem it solves, why choose it, and when not to use it |
| [RFC-0002 - Wire Format](RFC-0002.md) | What the byte structure looks like |
| [RFC-0003 - Conformance](RFC-0003.md) | How to verify the correctness of my implementation (test vectors) | English translations: [`en/RFC-0001.md`](../en/RFC-0001.md) · [`en/RFC-0002.md`](../en/RFC-0002.md) · [`en/RFC-0003.md`](../en/RFC-0003.md)

Homepage: [cyclone-protocol.github.io/cyclone](https://cyclone-protocol.github.io/cyclone/) - source code at [`index.html`](../index.html) (English) and [`vi/index.html`](index.html) (Vietnamese)

---

## Contributing

Cyclone is a technical specification, not a library. Implementations are needed for Rust, Go, C#/Unity, C/embedded, and Zig/C++.

The sole criterion for "Cyclone Compatible" status:

```
Run the test vectors in RFC-0003

↓

Achieve 100% success
```

There is no such thing as "98%". A single failed test vector implies a discrepancy between your implementation and another implementation.

Check the [ecosystem status table](https://cyclone-protocol.github.io/cyclone/#implementations) to see which languages ​​currently lack an implementation.

### Translations

Vietnamese is the original language; English is the translation.

---

## Repository Structure

```
.
├── README.md            This document (English)
├── LICENSE              CC BY 4.0
│
├── vi/                  Vietnamese - original version
│   ├── README.md
│   ├── RFC-0001.md      What is Cyclone
│   ├── RFC-0002.md      Wire Format Specification
│   └── RFC-0003.md      Conformance
│
└── en/                  English - translated version
├── RFC-0001.md
├── RFC-0002.md
└── RFC-0003.md
```

## License

This entire repository is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) - see the [`LICENSE`](../LICENSE) file.

You are free to copy, translate, quote, and create derivative works based on this specification, including for commercial purposes, provided that you give appropriate credit to the source/author.

This is a technical specification, not software. Your implementation is an independent work, not a derivative work of this specification - you are free to choose the license for it.

## Copyright

Copyright © 2026 Ha Duy Thang