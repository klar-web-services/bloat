# .bloat Binary Encoding Format

Version 0.1
Status: Draft Specification
File Extension: `.bloat`
Media Type: `application/x-bloat`

## 1. Introduction

`.bloat` is a binary encoding format that deliberately couples encoded size with computational decoding cost.

The format defines three encoding modes with progressively larger asymptotic representations:

$$
\begin{aligned}
\text{BLOAT-1} &: \Theta(n) \\
\text{BLOAT-2} &: \Theta(n^2) \\
\text{BLOAT-3} &: \Theta(2^n)
\end{aligned}
$$

where \(n\) is the source payload size in bits.

Unlike a conventional container format, `.bloat` does not store an ordinary payload alongside redundant auxiliary data. The expanded representation is itself the encoded payload. Reconstruction of the source depends on processing the representation defined by the selected BLOAT mode.

The format is intended as an experimental storage and computation format, a benchmarking target, and a basis for competitive decoder implementations.

---

## 2. Conformance Language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**, and **MAY** are to be interpreted as normative requirements.

An implementation conforms to `.bloat` version 0.1 if it:

1. correctly interprets the common file header;
2. implements at least one standard BLOAT mode;
3. reconstructs the original source bitstream exactly;
4. follows the encoding function specified for the declared mode; and
5. preserves the mandatory dependency between the encoded representation and the reconstructed payload.

---

## 3. Source Data Model

A `.bloat` source payload is an arbitrary finite bitstring:

$$
B=b_0,b_1,\ldots,b_{n-1}
$$

where:

$$
b_i\in\{0,1\}
$$

and \(n\) is the source length in bits.

Source bits are indexed from zero.

The encoded representation is mode-dependent and is defined separately for BLOAT-1, BLOAT-2, and BLOAT-3.

---

## 4. Design Properties

A `.bloat` encoding is designed around the following properties.

### 4.1 Exact Reconstruction

Decoding MUST reproduce the original bitstream exactly.

### 4.2 Mandatory Representation

Encoded units defined by a BLOAT mode participate directly in reconstruction of the source payload.

### 4.3 Complexity Coupling

The asymptotic size of a BLOAT mode corresponds to its asymptotic decoding workload.

### 4.4 Implementation Freedom

The specification defines logical encoding semantics rather than a required execution strategy.

Implementations MAY use:

* vectorized instructions;
* parallel CPU execution;
* GPU computation;
* FPGA or ASIC acceleration;
* distributed processing;
* memory mapping;
* direct storage access;
* custom storage layouts; and
* other equivalent optimizations.

### 4.5 Mode Independence

Each BLOAT mode is a complete source encoding.

Higher-numbered modes are not containers for lower-numbered modes.

---

# 5. File Structure

A `.bloat` file consists of:

1. a common header; and
2. a mode-specific encoded payload.

Conceptually:

```text
+--------------------------+
| Common Header            |
+--------------------------+
| Mode-Specific Payload    |
|                          |
|                          |
+--------------------------+
```

The common header is small relative to the encoded payload and does not contribute to the asymptotic storage classification of a BLOAT mode.

---

# 6. Common Header

Version 0.1 defines the following logical header fields.

| Field                | Description                             |
| -------------------- | --------------------------------------- |
| Magic                | Identifies a `.bloat` artifact          |
| Version              | Format version                          |
| Mode                 | BLOAT-1, BLOAT-2, or BLOAT-3            |
| Source bit length    | Number of bits in the original payload  |
| Source byte length   | Original byte length, when byte-aligned |
| Encoding identifier  | Mode-specific encoding identifier       |
| Payload offset       | Start of encoded payload                |
| Integrity identifier | Optional integrity mechanism            |

A canonical textual representation might contain:

```text
Magic: BLOAT
Version: 0.1
Mode: BLOAT-2
Source-Bits: 80000000
Encoding: PARITY-MATRIX-1
```

The serialized binary header layout MAY be standardized separately.

---

# 7. BLOAT-1

## 7.1 Definition

BLOAT-1 is the linear encoding mode.

Each source bit is represented by one parity-share block of:

$$
85\text{ KiB}
$$

which is:

$$
87,040\text{ bytes}
$$

or:

$$
696,320\text{ bits}
$$

Let the encoded block for source bit \(b_i\) be:

$$
S_i =
s_{i,0},
s_{i,1},
\ldots,
s_{i,696319}
$$

The source bit is defined by:

$$
\boxed{
b_i =
\bigoplus_{k=0}^{696319}s_{i,k}
}
$$

where \(\oplus\) denotes XOR.

---

## 7.2 Encoding

For each source bit \(b_i\):

1. The encoder selects values for the first 696,319 share bits.
2. It computes their XOR.
3. It selects the final share bit such that the parity of the complete block equals \(b_i\).
4. The full 85 KiB block is written to the encoded payload.

Blocks are stored in source-bit order unless another physical layout is explicitly declared.

---

## 7.3 Decoding

For each source bit \(b_i\), the decoder computes the parity of the corresponding 85 KiB block:

$$
b_i =
\operatorname{parity}(S_i)
$$

The resulting bits are concatenated in source order.

---

## 7.4 Encoded Size

For a source of \(n\) bits:

$$
\boxed{
S_1(n)=87,040n\text{ bytes}
}
$$

The storage complexity is:

$$
\boxed{\Theta(n)}
$$

The decoding complexity is:

$$
\boxed{\Theta(n)}
$$

with a fixed expansion factor of:

$$
696,320
$$

encoded bits per source bit.

---

# 8. BLOAT-2

## 8.1 Definition

BLOAT-2 is the quadratic encoding mode.

The encoded payload is an:

$$
n\times n
$$

matrix of fixed-size cells.

Each cell is:

$$
512\text{ bytes}
$$

or:

$$
4096\text{ bits}
$$

Let:

$$
C_{i,j}
$$

denote the cell at row \(i\), column \(j\).

Each row corresponds to one source bit.

Define the cell parity function:

$$
p(C_{i,j})
=
\bigoplus_{k=0}^{4095}
c_{i,j,k}
$$

The source bit \(b_i\) is:

$$
\boxed{
b_i =
\bigoplus_{j=0}^{n-1}
p(C_{i,j})
}
$$

---

## 8.2 Encoding

For each source bit \(b_i\):

1. The encoder constructs \(n-1\) cells of 512 bytes each.
2. It computes the XOR of their cell parities.
3. It constructs the final cell such that the parity across all \(n\) cells in row \(i\) equals \(b_i\).
4. The complete row is written to the encoded matrix.

The process is repeated for all \(n\) source bits.

---

## 8.3 Decoding

For each row \(i\):

1. Compute the parity of every cell \(C_{i,j}\).
2. XOR the resulting \(n\) cell parities.
3. Emit the result as source bit \(b_i\).

Formally:

$$
b_i =
\bigoplus_{j=0}^{n-1}
p(C_{i,j})
$$

Rows MAY be decoded independently and in parallel.

---

## 8.4 Encoded Size

The matrix contains:

$$
n^2
$$

cells.

Therefore:

$$
\boxed{
S_2(n)=512n^2\text{ bytes}
}
$$

Storage complexity:

$$
\boxed{\Theta(n^2)}
$$

Encoding complexity:

$$
\boxed{\Theta(n^2)}
$$

Decoding complexity:

$$
\boxed{\Theta(n^2)}
$$

---

# 9. BLOAT-3

## 9.1 Definition

BLOAT-3 is the exponential encoding mode.

For a source containing \(n\) bits, the encoded payload contains:

$$
\boxed{2^n}
$$

share bytes.

Let the encoded share sequence be:

$$
C_0,C_1,\ldots,C_{2^n-1}
$$

Each share byte contributes to one source-bit accumulator.

---

## 9.2 Share Assignment

Share byte \(C_j\) is assigned to source bit:

$$
\boxed{
a(j)=j\bmod n
}
$$

This partitions the encoded share space into \(n\) buckets.

---

## 9.3 Share Contribution

For byte \(C_j\), define:

$$
p(C_j)
$$

as the parity of its eight constituent bits.

Each source bit is defined as:

$$
\boxed{
b_i =
\bigoplus_{\substack{0\le j<2^n\\j\bmod n=i}}
p(C_j)
}
$$

The complete source is obtained by evaluating this reduction for every \(i\).

---

## 9.4 Encoding

For each source-bit bucket:

1. The encoder assigns values to all but one share byte in that bucket.
2. It computes the XOR of their byte parities.
3. It constructs the final share byte such that the bucket parity equals the corresponding source bit.
4. All share bytes are emitted in index order.

The resulting encoded payload contains exactly:

$$
2^n
$$

bytes.

---

## 9.5 Decoding

The decoder initializes an \(n\)-bit accumulator array to zero.

For each encoded byte \(C_j\):

1. Compute \(p(C_j)\).
2. Compute:

$$
i=j\bmod n
$$

3. XOR \(p(C_j)\) into accumulator \(i\).

After all share bytes have been processed, the accumulator array is the reconstructed source bitstream.

---

## 9.6 Encoded Size

$$
\boxed{
S_3(n)=2^n\text{ bytes}
}
$$

Storage complexity:

$$
\boxed{\Theta(2^n)}
$$

Encoding complexity:

$$
\boxed{\Theta(2^n)}
$$

Decoding complexity:

$$
\boxed{\Theta(2^n)}
$$

---

# 10. Mode Summary

| Mode    | Encoding Structure                            |      Encoded Size | Encode Complexity | Decode Complexity |
| ------- | --------------------------------------------- | ----------------: | ----------------: | ----------------: |
| BLOAT-1 | 85 KiB parity block per source bit            | \(87,040n\) bytes |     \(\Theta(n)\) |     \(\Theta(n)\) |
| BLOAT-2 | \(n\times n\) matrix of 512-byte parity cells |  \(512n^2\) bytes |   \(\Theta(n^2)\) |   \(\Theta(n^2)\) |
| BLOAT-3 | Exponential byte-share space                  |     \(2^n\) bytes |   \(\Theta(2^n)\) |   \(\Theta(2^n)\) |

---

# 11. Physical Storage

The logical encoding defined by this specification is independent of physical storage architecture.

A conforming `.bloat` implementation MAY store the encoded payload using:

* a contiguous file;
* multiple files;
* block storage;
* object storage;
* database-backed storage;
* distributed storage;
* memory-mapped regions;
* striped devices; or
* specialized hardware.

The physical representation MUST preserve the logical encoded values defined by the selected BLOAT mode.

---

# 12. Decoder Implementation

A decoder MAY process encoded units in any order permitted by the selected encoding.

A decoder MAY:

* process multiple blocks, rows, cells, or shares concurrently;
* perform hierarchical XOR reductions;
* use SIMD instructions;
* dispatch work to GPUs or other accelerators;
* partition work across machines;
* cache intermediate parity results;
* decode source regions on demand where the mode permits it;
* overlap storage I/O with computation; and
* employ source-independent preprocessing.

The specification intentionally leaves these implementation choices open.

---

# 13. Streaming Characteristics

## 13.1 BLOAT-1

BLOAT-1 naturally supports source-bit and source-range streaming.

A decoder may reconstruct source bit \(i\) after processing its corresponding block.

## 13.2 BLOAT-2

BLOAT-2 supports row-level streaming.

Source bit \(i\) becomes available after row \(i\) has been reduced.

Rows may be processed in parallel or in source order.

## 13.3 BLOAT-3

BLOAT-3 supports incremental accumulator updates while scanning the share space.

A complete source bit is finalized only when all shares assigned to its bucket have been accounted for.

---

# 14. Integrity

An implementation MAY provide an integrity mechanism in addition to the core BLOAT encoding.

Suitable mechanisms include:

* whole-file cryptographic hashes;
* block hashes;
* Merkle trees;
* per-region checksums; and
* authenticated storage layers.

Integrity metadata is not part of the source reconstruction function defined by version 0.1.

Integrity metadata SHOULD be designed so that it does not alter the size class of the selected BLOAT mode.

---

# 15. Byte Alignment

When the source payload is byte-aligned, the header SHOULD contain both:

$$
n
$$

and:

$$
n/8
$$

as the source bit and byte lengths respectively.

For non-byte-aligned inputs, the final decoded byte MUST define the number of significant bits through the source-length field.

---

# 16. Determinism

`.bloat` encoding is generally non-deterministic.

Multiple valid `.bloat` artifacts MAY encode the same source payload because share values may be selected arbitrarily subject to the parity constraints of the active mode.

A canonical encoder MAY define a deterministic share-generation process for reproducibility.

If deterministic generation is used, the generated shares remain part of the encoded representation.

---

# 17. Reference Size Example

Consider a source of:

$$
200\text{ GB}
$$

using decimal gigabytes.

The source contains:

$$
200\times10^9\text{ bytes}
$$

or:

$$
n=1.6\times10^{12}\text{ bits}
$$

---

## 17.1 BLOAT-1

$$
S_1(n)=87,040n
$$

Therefore:

$$
S_1=
87,040\times1.6\times10^{12}
$$

$$
=
1.39264\times10^{17}\text{ bytes}
$$

which is approximately:

$$
\boxed{139.3\text{ PB}}
$$

---

## 17.2 BLOAT-2

$$
S_2(n)=512n^2
$$

Therefore:

$$
S_2=
512(1.6\times10^{12})^2
$$

$$
=
1.31072\times10^{27}\text{ bytes}
$$

Using:

$$
1\text{ RB}=10^{27}\text{ bytes}
$$

the encoded size is:

$$
\boxed{1.31\text{ RB}}
$$

---

## 17.3 BLOAT-3

$$
S_3(n)=2^n
$$

For:

$$
n=1.6\times10^{12}
$$

the encoded size is:

$$
2^{1.6\times10^{12}}\text{ bytes}
$$

Using:

$$
\log_{10}(2^{1.6\times10^{12}})
=
1.6\times10^{12}\log_{10}(2)
$$

gives an approximate magnitude of:

$$
\boxed{
2.34\times10^{481,647,993,062}
\text{ bytes}
}
$$

---

# 18. Reference Decoder Model

A generic decoder may be represented as:

```text
read common header

switch mode:

    BLOAT-1:
        for each source bit:
            reduce corresponding 85 KiB block by XOR
            emit resulting bit

    BLOAT-2:
        for each source row:
            reduce each 512-byte cell by XOR
            reduce all cell parities by XOR
            emit resulting bit

    BLOAT-3:
        allocate n source-bit accumulators

        for each encoded share byte:
            compute byte parity
            map share index to source accumulator
            XOR parity into accumulator

        emit accumulator array
```

This model is normative only with respect to the mathematical result. Implementations MAY use any equivalent computation.

---

# 19. Competitive Decoding Profile

`.bloat` is suitable for comparative decoder benchmarking because conforming implementations operate on the same logical representation while retaining broad freedom in execution strategy.

A competition profile MAY specify:

* BLOAT mode;
* source payload;
* encoded artifact;
* permitted hardware;
* permitted preprocessing;
* storage topology;
* cache policy;
* maximum memory usage;
* time-to-first-output measurement;
* sustained decode throughput; and
* application-level performance metrics.

For executable workloads such as video games, useful measurements MAY include:

* time to process start;
* time to first rendered frame;
* average frame rate;
* frame-time distribution;
* total hardware cost;
* peak power consumption; and
* decoded working-set size.

Competition profiles are outside the core `.bloat` file-format specification.

---

# 20. Security and Resource Considerations

`.bloat` files may have extremely large resource requirements.

Implementations SHOULD validate the header and compute the expected encoded size before allocating resources or initiating a decode.

Decoders SHOULD support configurable limits for:

* encoded size;
* source size;
* memory allocation;
* processing time;
* storage reads;
* concurrent tasks; and
* accelerator usage.

Implementations processing untrusted `.bloat` artifacts SHOULD treat declared source length and mode as resource-sensitive input.

BLOAT-3 artifacts in particular may specify encoded representations that are not physically realizable with available storage systems.

---

# 21. Versioning

The format version is independent of the BLOAT mode.

For example:

```text
.bloat version 0.1, BLOAT-1
.bloat version 0.1, BLOAT-2
.bloat version 0.1, BLOAT-3
```

Future format versions MAY introduce:

* additional BLOAT modes;
* additional cell functions;
* alternate share widths;
* standardized physical layouts;
* standardized integrity structures;
* stream-oriented profiles;
* benchmark profiles; and
* application-specific extensions.

A future version SHOULD preserve explicit identification of both format version and BLOAT mode.

---

# 22. Summary

`.bloat` version 0.1 defines three binary encoding modes in which representation size and reconstruction cost scale together.

BLOAT-1 uses large parity blocks to produce substantial linear expansion.

BLOAT-2 uses a quadratic matrix of parity cells.

BLOAT-3 uses an exponential share space.

The format specifies the mathematical relationship between source bits and encoded data while leaving decoder architecture open to optimization.

The resulting progression is:

$$
\boxed{
\begin{aligned}
\text{BLOAT-1} &: 87,040n\text{ bytes} \\
\text{BLOAT-2} &: 512n^2\text{ bytes} \\
\text{BLOAT-3} &: 2^n\text{ bytes}
\end{aligned}
}
$$

with corresponding decode complexities:

$$
\boxed{
\Theta(n),\quad
\Theta(n^2),\quad
\Theta(2^n)
}
$$

This relationship between encoded size and required reconstruction work is the defining property of the `.bloat` format.
