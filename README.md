# .bloat

A binary encoding format that stores data as inefficiently as possible.

`.bloat` encodes arbitrary binary data at the largest expansion factor that
still permits exact reconstruction. Wasting space is the goal, not a side
effect. The three modes exist so the waste can be scaled from merely
impractical up to physically impossible.

Padding would not achieve this, because a decoder could skip it. Instead the
expansion carries the payload: the source bitstream is recovered only by
XOR-reducing the whole artifact, so every byte written has to be read back.

Normative definition: [`spec.md`](spec.md) (version 0.1, draft).

## Design goal

What the format optimizes for:

- Encoded size, maximized.
- Mandatory work. No encoded unit may be skippable during reconstruction.
- Exact reconstruction, bit for bit. Waste only counts if the artifact still
  decodes.

Throughput, storage cost and latency are stated non-goals (spec §1). Making a
`.bloat` artifact smaller or cheaper to decode, without buying exactness, is a
regression.

Expansion is normative. An encoder must emit the full representation its mode
defines and may not substitute a compacted one (spec §4.6). Compression in a
storage layer underneath the logical representation is allowed, but the logical
size is the size that counts, and implementations relying on such a layer
should report both.

## Modes

For a source of `n` bits:

| Mode | Structure | Encoded size | Encode | Decode |
| --- | --- | ---: | ---: | ---: |
| BLOAT-1 | one 85 KiB parity block per source bit | `87,040n` bytes | Θ(n) | Θ(n) |
| BLOAT-2 | `n × n` matrix of 512-byte parity cells | `512n²` bytes | Θ(n²) | Θ(n²) |
| BLOAT-3 | `2ⁿ` share bytes bucketed by `j mod n` | `2ⁿ` bytes | Θ(2ⁿ) | Θ(2ⁿ) |

Modes are independent complete encodings. Higher modes do not wrap lower ones
(spec §4.5). BLOAT-1 spends a fixed 696,320 encoded bits per source bit.
BLOAT-2 makes the spend per source bit grow with the source. BLOAT-3 exceeds
any available storage system for all but very short inputs.

Reconstruction rules:

```
BLOAT-1:  b_i = parity(S_i)                            # S_i = 696,320 bits
BLOAT-2:  b_i = XOR_{j<n} parity(C_i,j)                # C_i,j = 4,096 bits
BLOAT-3:  b_i = XOR_{j<2^n, j mod n = i} parity(C_j)   # C_j = 8 bits
```

## File layout

```
+--------------------------+
| Common Header            |   magic, version, mode, source bit/byte length,
+--------------------------+   encoding id, payload offset, integrity id
| Mode-Specific Payload    |
+--------------------------+
```

The header is asymptotically negligible, which makes it the only compact part
of the format. Its binary serialization is not fixed in 0.1 (spec §6).

## Conformance

An implementation conforms to 0.1 if it (spec §2):

1. interprets the common header;
2. implements at least one standard mode;
3. reconstructs the source bitstream exactly;
4. follows the declared mode's encoding function;
5. preserves the dependency between encoded representation and payload; and
6. emits the complete representation its mode requires, where it encodes.

Encoding is non-deterministic. Share bits are unconstrained apart from the
parity condition, so one source has many valid encodings (spec §16).

## Implementation notes

The spec fixes the mathematics, not the execution strategy. SIMD, GPU and FPGA
offload, distributed reduction, memory mapping and custom physical layouts are
all permitted, provided the logical encoded values survive (spec §4.4, §11,
§12). The mode fixes the workload, so implementations compete on how quickly
they get through it rather than on how much of it they can avoid (spec §19).

Decoders reading untrusted artifacts should validate the header and compute the
expected encoded size before allocating anything. A 30-byte header can request
an artifact larger than the observable universe (spec §20).

## Scale reference

Encoded size of a 200 GB source (`n = 1.6 × 10¹²` bits), per spec §17:

| Mode | Encoded size |
| --- | --- |
| BLOAT-1 | ~139.3 PB |
| BLOAT-2 | ~1.31 RB (`10²⁷` bytes) |
| BLOAT-3 | ~`2.34 × 10^481,647,993,062` bytes |

BLOAT-2 exceeds total world storage capacity (~`10²³` bytes) by about four
orders of magnitude. BLOAT-3 exceeds the particle count of the observable
universe (~`10⁸⁰`) by roughly 481 billion orders of magnitude. All three are
conforming encodings of the same 200 GB.

## Status

Draft specification. No reference encoder or decoder yet.

This file is 4,676 bytes. Encoded as BLOAT-1 it would be 3.3 GB; as BLOAT-2, 716 GB.
BLOAT-3 is left as an exercise.

Media type: `application/x-bloat`. Extension: `.bloat`.
