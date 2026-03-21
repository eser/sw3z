# SW3Z — SW³ Zipped Archive Format

SW3Z is a read-optimized game asset archive format for id Tech 3 derived
engines. It is a modernized replacement for the `.pk3` container format,
designed around the realities of contemporary hardware: NVMe storage,
Vulkan/D3D12 GPU pipelines, high-resolution texture sets, and per-file
streaming.

---

## Quick Reference

| Property            | Value                                  |
| ------------------- | -------------------------------------- |
| Magic               | `SW3Z` (0x53 0x57 0x33 0x5A)           |
| Index position      | Start of file                          |
| Default compression | LZ4 Frame Format                       |
| Checksum            | CRC32C (Castagnoli)                    |
| Signing             | Ed25519 (optional)                     |
| Entry size          | 40 bytes                               |
| Max archives        | Unlimited (stack/override by priority) |

See [SPEC.md](./SPEC.md) for the full format specification.

---

## Rationale

### Why not just use pk3?

The `.pk3` format is a renamed ZIP file using DEFLATE compression. It has served
the Quake modding community well since 1999, but it shows its age in several
ways:

- **DEFLATE is slow to decompress** — roughly 300–500 MB/s on a single core.
  Modern games load dozens of assets per frame; decompression becomes a CPU
  bottleneck.
- **Central Directory is at the end** — a reader must seek to the end of the
  file to discover what is inside, then seek back to read any asset. On local
  storage this costs two seeks per archive open.
- **No per-file compression convention** — pk3 tooling applies DEFLATE
  uniformly. Already-compressed assets (JPEG, OGG, DXT) are re-compressed for no
  benefit, wasting CPU at load time.
- **No alignment guarantees** — Vulkan and D3D12 require texture data to be
  aligned to specific boundaries (256 B, 4 KiB) for direct GPU upload. ZIP
  provides no mechanism for this.
- **Weak runtime integrity** — ZIP includes per-file CRC32, but pk3 loaders
  rarely validate it at load time. Corruption surfaces as visual glitches or
  crashes with no indication of which file is affected.

SW3Z addresses all of these while preserving what made pk3 great: a simple
stackable archive system where later archives override earlier ones, enabling
mods and patches without touching base content.

---

### Container format: why not TAR?

TAR and ZIP are the two obvious starting points. The comparison came down to one
question: **where is the index?**

TAR has no index at all. File headers are inline, immediately before each file's
data. Finding a specific asset requires a linear scan from the beginning — O(n).
For a game engine that needs to load a texture 60 times per second, this is
unacceptable.

ZIP has a Central Directory at the _end_ of the file. This was a deliberate 1989
design choice to enable sequential single-pass writing — you write files as you
go, then append the directory at the end. For network partial downloads, this
also allows fetching just the directory via an HTTP Range request, then
selectively downloading only needed files.

SW3Z takes the best of both and discards the rest:

- **Index at the start**, like neither TAR nor ZIP — enabling a single seek to
  load the full index on archive open.
- **Per-file metadata** from ZIP — every entry knows its own offset, sizes, and
  checksum without scanning.
- **Simple flat layout** from TAR — no ZIP spec complexity, no local vs. central
  header duplication.

The tradeoff is that the writer must know all offsets before writing the header.
In practice this means a two-pass write or building the index in memory. Since
SW3Z is written offline by Go tooling (not streamed), this is a non-issue.

---

### Compression: why LZ4?

Our specific scenario is important here: **assets are converted from pk3 on the
player's local machine at install time**, not downloaded in the SW3Z format.
This changes the compression calculus significantly.

| Algorithm      | Decompress  | Compress     | Ratio     | License     |
| -------------- | ----------- | ------------ | --------- | ----------- |
| DEFLATE (zlib) | ~400 MB/s   | slow         | good      | free        |
| LZ4            | ~4,000 MB/s | fast         | moderate  | free        |
| zstd           | ~1,500 MB/s | configurable | very good | free        |
| Oodle Kraken   | ~1,800 MB/s | slow         | excellent | proprietary |

Because compression happens once at install time, compress speed is irrelevant.
Only **decompress speed** and **disk footprint** matter at runtime.

LZ4 was chosen because:

1. **Decompress speed is unmatched among free algorithms** — ~4 GB/s approaches
   RAM bandwidth. This means the CPU is almost never the bottleneck when
   streaming assets; the GPU pipeline or disk I/O will saturate first.
2. **Background streaming is central to the engine** — with many small assets
   loading across frames, LZ4's near-zero CPU overhead keeps the decompression
   thread from interfering with render and simulation threads.
3. **No patents, no license fees** — BSD 2-Clause. The same author (Yann Collet,
   now at Meta) also wrote zstd.
4. **LZ4 Frame Format is binary-compatible** between Go
   (`github.com/pierrec/lz4/v4`) and C (`lz4frame.h`), with no glue code
   required.

**What about zstd?** zstd is also free, also patent-free, and decompresses at
~1,500 MB/s — fast enough for almost any scenario. It is included in the spec as
compression algorithm `0x02` for cases where disk space is a concern (e.g., very
large texture sets). Writers may choose zstd per-file; the engine handles both
transparently. Think of LZ4 as the performance default and zstd as the
space-saving option.

**What about Oodle?** Excellent ratios and speeds, but requires a commercial
license outside of Unreal Engine. Not appropriate for an open-source format.

---

### Checksum: why CRC32C and not xxHash or SHA-256?

Several algorithms were considered for the per-entry integrity checksum.

**SHA-256** was rejected immediately — it is a cryptographic hash, slow by
design, and the threat it defends against (intentional collision) is already
covered by Ed25519 archive signing. Using SHA-256 for a corruption check is
using a sledgehammer to hang a picture.

**xxHash32 / xxHash64** are excellent non-cryptographic hashes — fast,
well-distributed, and widely used. However, Go's standard library has no xxHash
support, requiring an external dependency for what is ultimately a simple
corruption check.

**CRC32C (Castagnoli)** was chosen for three reasons:

1. **Zero additional dependencies** — `hash/crc32` is in Go's standard library.
   In C, hardware intrinsics (x86 SSE4.2 `_mm_crc32_u*`, ARM `__crc32c*`)
   provide single-instruction implementations; lightweight software fallbacks
   exist for older CPUs.
2. **Hardware acceleration is universal** — on modern CPUs, CRC32C runs at
   near-memcpy speed when using SSE4.2 or ARM CRC instructions. Developers who
   need maximum throughput can drop in the hardware path with no spec changes.
3. **Consistency with LZ4** — the LZ4 Frame Format uses CRC32C for its own
   internal content checksum. Using the same polynomial across both layers
   avoids having two different checksum algorithms in the same code path.

CRC32C is not cryptographic and cannot detect intentional tampering — nor does
it need to. That responsibility belongs to the Ed25519 archive signature.
Per-entry corruption detection uses CRC32C instead — a much better fit for the
task.

---

### Archive signing: why Ed25519?

Archive-level integrity (CRC32C per entry) answers "was this file corrupted?"
Ed25519 signing answers a different question: "did a trusted party produce this
archive?"

This matters for:

- **Anti-cheat** — a server can refuse to load unsigned or incorrectly signed
  archives in competitive play.
- **Official content verification** — distinguishing first-party releases from
  community modifications.
- **Tamper detection** — detecting post-distribution modification of an archive.

Ed25519 was chosen over ECDSA P-256 or RSA because:

- 64-byte signatures — minimal EOF overhead
- Fast verification — relevant when loading many archives at startup
- Small key sizes — 32-byte public key, easily embedded in engine config
- Modern, well-audited — no known practical attacks

SHA-256 was explicitly rejected as a signing method. A hash without a private
key proves nothing about authorship — anyone can recompute the same hash.

Public key distribution is intentionally out of scope. Engine integrations are
expected to embed or configure trusted public keys independently.

---

### Index structure: why FNV-1a for path hashing?

Path lookup at runtime must be fast. String comparison across thousands of
entries on every asset load would be unacceptable. A hash map keyed on a 64-bit
path hash reduces lookup to O(1) with a single integer comparison in the common
case.

FNV-1a 64-bit was chosen because:

- Trivially implementable in ~10 lines of C with no dependencies
- The same hash is already used elsewhere in the codebase for path normalization
- 64-bit output makes accidental collisions astronomically unlikely in practice
- In the rare case of a collision, the spec mandates fallback to full string
  comparison via the string table

Paths are normalized to lowercase with forward slashes before hashing. Directory
paths include the trailing `/` — `textures/gothic/` and `textures/gothic` are
distinct entries with distinct hashes. This rule is explicit in the spec to
prevent silent cross-implementation incompatibilities.

---

### Alignment: why power-of-2 exponent?

Modern GPU APIs (Vulkan, D3D12) and NVMe direct I/O require asset data to begin
at specific byte boundaries. Rather than a fixed global alignment, SW3Z supports
per-entry alignment as a power-of-2 exponent stored in one byte.

```c
uint32_t alignment = 1u << entry.alignment_exp;  // single instruction
```

The exponent encoding was preferred over an enum for a simple reason: it
eliminates the lookup table. Any power-of-2 alignment is representable, and
decoding is a single bit-shift. Valid exponents range from 4 (16 bytes, SIMD
minimum) to 16 (64 KiB, large page / GPU texture alignment). Values outside this
range are treated as malformed entries.

Padding bytes are zeroed and not counted in `compressed_size`. Readers MUST use
`data_offset` to locate assets — never assume contiguous layout.

---

### What was deliberately left out

Several features were proposed and rejected for v1.0:

- **File timestamps** — no use case in a game engine; asset freshness is
  determined by archive priority order, not modification time.
- **Directory entries as required** — directories are implicit from path
  strings. Explicit directory entries are supported but optional.
- **SHA-256 as a signing option** — conflates integrity with authenticity;
  removed in favor of the cleaner CRC32C + Ed25519 separation.
- **Arbitrary compression enums** — only three values defined (none, LZ4, zstd).
  Unknown values cause the entry to be skipped, not the archive to be rejected.
- **Content-addressable storage** — asset aliasing (multiple paths pointing to
  the same `data_offset`) covers the primary deduplication use case without the
  complexity of a full CAS layer.

A machine-readable format description (Kaitai Struct `.ksy`) is planned
post-v1.0 finalization.

---

## Contributing

See [CONTRIBUTING.md](./.github/CONTRIBUTING.md) for guidelines.

## License

[Apache-2.0](./LICENSE)

---

_SW3Z named after [SW³](https://software3.org)._
