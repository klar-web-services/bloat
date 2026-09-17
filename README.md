# .bloat

A binary encoding format whose design goal is maximal storage inefficiency.

`.bloat` stores arbitrary binary data at the largest expansion factor its
authors could specify while keeping the encoding well-defined and exactly
reversible. Space waste is the objective, not a side effect of some other
property: the format exists to consume storage, and the mode progression
(linear → quadratic → exponential) exists so that the waste can be dialled up
past the point of physical realizability.

Decoding cost is bound to that expansion by construction. `.bloat` is not a
container that pads a payload with filler, because filler could be skipped —
instead the expanded representation *is* the payload, and the source bitstream
exists only as the result of an XOR reduction over the entire encoded artifact.
Every wasted byte must be read and reduced to recover the data it encodes.

Full normative definition: [`spec.md`](spec.md) (version 0.1, draft).

## Design goal

The format optimizes for:

- **Encoded size** — bytes written per source bit, maximized.
- **Mandatory work** — no encoded unit may be skippable during reconstruction.
- **Exact reconstruction** — the output must be bit-identical to the input;
  waste is only interesting if the artifact is still a faithful encoding.

It explicitly does not optimize for throughput, storage cost, latency, or any
conventional measure of encoder quality — these are stated non-goals (spec §1).
A change that makes `.bloat` smaller or cheaper to decode, without buying
exactness, is a regression.

Expansion is normative, not incidental: an encoder must emit the full
representation its mode defines and may not substitute a compacted one
(spec §4.6). Compaction in a storage layer beneath the logical representation
is permitted, but the logical size remains the conforming size of the artifact.

## Modes

For a source of `n` bits:

| Mode | Structure | Encoded size | Encode | Decode |
| --- | --- | ---: | ---: | ---: |
| BLOAT-1 | one 85 KiB parity block per source bit | `87,040n` bytes | Θ(n) | Θ(n) |
| BLOAT-2 | `n × n` matrix of 512-byte parity cells | `512n²` bytes | Θ(n²) | Θ(n²) |
| BLOAT-3 | `2ⁿ` share bytes bucketed by `j mod n` | `2ⁿ` bytes | Θ(2ⁿ) | Θ(2ⁿ) |

Modes are independent complete encodings; higher modes do not wrap lower ones
(spec §4.5). Selecting a mode selects an expansion class: BLOAT-1 wastes a
fixed 696,320 encoded bits per source bit, BLOAT-2 makes the waste per source
bit grow with the source, and BLOAT-3 makes the artifact exceed any available
storage system for all but trivially short inputs.

Reconstruction rules:

```
BLOAT-1:  b_i = parity(S_i)                       # S_i = 696,320 bits
BLOAT-2:  b_i = XOR_{j<n} parity(C_i,j)           # C_i,j = 4,096 bits
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

The header is asymptotically negligible. Its binary serialization is not fixed
in 0.1 (spec §6).

## Conformance

An implementation conforms to 0.1 if it (spec §2):

1. interprets the common header;
2. implements at least one standard mode;
3. reconstructs the source bitstream exactly;
4. follows the declared mode's encoding function; and
5. preserves the dependency between encoded representation and payload.

Encoding is non-deterministic — share bits are free apart from the parity
constraint, so many valid artifacts encode the same source (spec §16).

## Implementation notes

The spec fixes the mathematics, not the execution strategy. SIMD, GPU/FPGA
offload, distributed reduction, memory mapping, and custom physical layouts are
all permitted as long as logical encoded values are preserved (spec §4.4, §11,
§12).

Decoders handling untrusted artifacts should validate the header and compute
the expected encoded size *before* allocating anything; BLOAT-3 routinely
declares sizes that are not physically realizable (spec §20).

## Scale reference

Encoded size of a 200 GB source (`n = 1.6 × 10¹²` bits), per spec §17:

- BLOAT-1 → ~139.3 PB
- BLOAT-2 → ~1.31 RB (`10²⁷` bytes)
- BLOAT-3 → ~`2.34 × 10^481,647,993,062` bytes

The BLOAT-2 figure exceeds total world storage capacity (~10²³ bytes) by some
four orders of magnitude. The BLOAT-3 figure exceeds the particle count of the
observable universe (~10⁸⁰) by roughly 481 billion orders of magnitude. Both
are conforming encodings of the same 200 GB.

## Status

Draft specification only. No reference encoder or decoder in this repository yet.

Media type: `application/x-bloat` · Extension: `.bloat`
