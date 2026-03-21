# SW3Z Format Specification
**Version:** 0.9
**Compression:** LZ4, Zstandard, or uncompressed (per-file)
**Endianness:** Little-endian throughout

> See [README.md](./README.md) for design rationale and motivation.

---

## Overview

SW3Z is a read-optimized game asset archive format. It is designed to replace pk3/zip containers in id Tech 3 derived engines. Key design goals:

- **Index-first:** The full file index is at the start of the archive, enabling O(1) random access with a single seek.
- **Per-file compression:** Each asset is independently compressed (or stored uncompressed), so loading one file does not require decompressing others. The compression algorithm is specified per-file.
- **Simple implementation:** The format can be implemented in ~300 lines of C with liblz4 (and optionally libzstd).
- **Stack/override support:** Multiple `.sw3z` archives load in priority order; later-loaded files override earlier ones by path hash.

---

## File Layout

```
┌─────────────────────────────────┐
│ File Header            (24 B)   │
├─────────────────────────────────┤
│ Index Entries          (N × 40B)│
├─────────────────────────────────┤
│ String Table           (variable)│
├─────────────────────────────────┤
│ Asset Data             (variable)│
│   [compressed/raw frames]       │
│   [alignment padding if needed] │
├─────────────────────────────────┤
│ Ed25519 Signature  (64 B, opt)  │
└─────────────────────────────────┘
```

---

## File Header (24 bytes)

| Offset | Size | Type     | Field           | Description                          |
|--------|------|----------|-----------------|--------------------------------------|
| 0      | 4    | u8[4]    | magic           | `S` `W` `3` `Z` (0x53 0x57 0x33 0x5A) |
| 4      | 2    | u16      | version         | Format version. Currently `1`.       |
| 6      | 2    | u16      | flags           | Bits 0–1: signing method. Bits 2–15: reserved, must be `0`. |
| 8      | 4    | u32      | entry_count     | Number of index entries.             |
| 12     | 4    | u32      | string_table_size | Byte length of the string table.   |
| 16     | 8    | u64      | data_offset     | Byte offset where asset data begins. |

An archive with `entry_count = 0` is valid and contains no assets.

---

## Index Entry (40 bytes each)

Immediately follows the file header. There are `entry_count` entries.

| Offset | Size | Type  | Field             | Description                                      |
|--------|------|-------|-------------------|--------------------------------------------------|
| 0      | 8    | u64   | path_hash         | FNV-1a 64-bit hash of the normalized path string.|
| 8      | 4    | u32   | string_offset     | Byte offset into the string table for full path. |
| 12     | 4    | u32   | string_length     | Byte length of path string (no null terminator). |
| 16     | 8    | u64   | data_offset       | Byte offset from start of file to asset data.     |
| 24     | 4    | u32   | compressed_size   | Byte size of the asset data on disk.              |
| 28     | 4    | u32   | uncompressed_size | Original byte size after decompression.          |
| 32     | 4    | u32   | crc32c            | CRC32C (Castagnoli) checksum of the uncompressed data. |
| 36     | 1    | u8    | compression       | Compression algorithm enum (see below).          |
| 37     | 1    | u8    | flags             | Per-file attribute bits (see below).             |
| 38     | 1    | u8    | alignment         | Power-of-2 exponent for data alignment (see below). |
| 39     | 1    | u8    | reserved          | Must be `0`.                                     |

---

## Compression Algorithms

| Value | Name          | Description                                      |
|-------|---------------|--------------------------------------------------|
| 0x00  | Uncompressed  | Raw data, no compression. `compressed_size` equals `uncompressed_size`. |
| 0x01  | LZ4 Frame     | LZ4 Frame Format (`LZ4F_*` API). Default.        |
| 0x02  | Zstandard     | Zstandard frame format. Better ratio, slower decompression. |

Readers that encounter an unsupported compression value MUST treat the entry as unreadable and MAY report a warning. The entry's metadata (path, size, checksum) remains valid for enumeration purposes.

---

## Entry Flags

| Bit | Name       | Description                                                              |
|-----|------------|--------------------------------------------------------------------------|
| 0   | executable | File has executable permission (`+x`).                                   |
| 1   | symlink    | Entry is a symbolic link. Data is the UTF-8 target path (uncompressed).  |
| 2   | aligned    | Asset data is aligned to `2^alignment` bytes (see Data Alignment).       |
| 3–7 | reserved   | Must be `0`.                                                             |

When `symlink` is set, the entry's asset data contains the raw UTF-8 target path string. `compression` must be `0x00` (uncompressed) and `compressed_size` must equal `uncompressed_size`. The `executable` and `aligned` flags are ignored when `symlink` is set. Writers SHOULD clear bits 0 and 2 when setting the symlink bit.

Directories are identified by a trailing `/` in the path string. Directory entries must have `data_offset = 0`, `compressed_size = 0`, `uncompressed_size = 0`, and `crc32c = 0`.

---

## Data Alignment

When the `aligned` flag (bit 2) is set, the `alignment` field (u8 at offset 38) specifies the alignment as a power-of-2 exponent: actual alignment = `2^alignment` bytes. Valid values range from `4` (16 bytes) to `16` (64 KiB). Values outside this range are malformed.

The writer inserts zero-byte padding before the asset data so that `data_offset` is a multiple of the alignment. Padding bytes are not counted in `compressed_size`.

When the `aligned` flag is clear, the `alignment` field must be `0x00`.

Common alignment values:

| N  | Alignment | Typical use case                         |
|----|-----------|------------------------------------------|
| 4  | 16 B      | SIMD (SSE, NEON)                         |
| 6  | 64 B      | Cache line, GPU minimum alignment        |
| 8  | 256 B     | Vulkan `minTexelBufferOffsetAlignment`   |
| 9  | 512 B     | NVMe logical sector                      |
| 12 | 4 KiB     | Page size, `mmap()`, NVMe direct I/O     |
| 16 | 64 KiB    | Large page, GPU texture alignment        |

---

## String Table

Immediately follows the index entries. A flat byte array of concatenated UTF-8 path strings. Strings are **not** null-terminated; length is stored in the index entry. Paths use forward slashes and are lowercase:

```
maps/q3dm1.bsp
textures/gothic/wall01.png
textures/gothic/
models/players/sarge/head.md3
```

---

## Asset Data

Immediately follows the string table (aligned to `data_offset` from the header). Each asset is stored according to its `compression` field in the index entry. For LZ4 (`0x01`), the data is an independent **LZ4 Frame Format** stream (`LZ4F_*` API — not raw LZ4 block), which includes its own magic number and content checksum for a second layer of integrity verification. For Zstandard (`0x02`), the data is a standard Zstandard frame, which is self-delimiting and includes an optional content checksum. For uncompressed (`0x00`), the data is stored as-is and `compressed_size` must equal `uncompressed_size`.

Assets are stored in index order. Alignment padding may insert zero-byte gaps between frames. Readers MUST use the per-entry `data_offset` to locate each asset rather than assuming contiguous layout.

---

## Path Hashing (FNV-1a 64-bit)

```c
uint64_t fnv1a_64(const char *path) {
    uint64_t hash = 0xcbf29ce484222325ULL;
    while (*path) {
        uint8_t c = (uint8_t)(*path++);
        if (c >= 'A' && c <= 'Z') c += 32; // lowercase
        if (c == '\\') c = '/';            // normalize separator
        hash ^= c;
        hash *= 0x00000100000001B3ULL;
    }
    return hash;
}
```

Normalize paths to lowercase with forward slashes before hashing and before writing to the string table. Directory paths include the trailing `/` in the hash — `textures/gothic/` and `textures/gothic` produce distinct path hashes.

---

## Stack / Override Behavior

Archives are loaded in priority order (e.g. numbered or by load list). When the engine resolves a path:

1. Build a hash map of `path_hash → (archive_index, entry_index)` across all loaded archives.
2. For duplicate hashes, the **last loaded** archive wins (higher priority overrides lower).
3. Collision: if two different paths produce the same hash, fall back to string comparison using the string table.

---

## Integrity

| Layer         | Mechanism                                          |
|---------------|----------------------------------------------------|
| Per-asset     | CRC32C stored in index, verified after decompress   |
| LZ4 frame     | LZ4 content checksum (enabled in frame prefs)      |
| Zstd frame    | Zstandard content checksum (optional in frame)      |
| Archive-level | Optional: Ed25519 signature of bytes 0..N-64 at EOF|

Archive-level signing is indicated by header `flags` bits 0–1.

**Why CRC32C (Castagnoli)?** CRC32C was chosen for three reasons: (1) it is universally available across platforms and languages with no additional dependencies, (2) hardware acceleration is available on both x86 (SSE4.2 `crc32` instruction) and ARM (ARMv8-A CRC extension), providing near-zero overhead in the decompression hot path, and (3) LZ4 Frame Format already uses CRC32C for its internal content checksum, so using the same variant for the per-entry checksum maintains consistency across integrity layers.

Per-entry CRC32C guards against unintended data corruption (bit rot, truncated writes, bad sectors). Archive-level integrity can be verified with external checksum files. For authenticity verification, use Ed25519 archive signing.

CRC32C uses the Castagnoli polynomial (`0x1EDC6F41`). Hardware-accelerated implementations are available via x86 SSE4.2 and ARMv8-A CRC extensions.

- **Go:** `crc32.New(crc32.MakeTable(crc32.Castagnoli))`
- **C (x86):** `_mm_crc32_u8` / `_mm_crc32_u32` / `_mm_crc32_u64`
- **C (ARM):** `__crc32cb` / `__crc32cw` / `__crc32cd`

---

## Archive Signing

| Value | Method   | EOF payload                                |
|-------|----------|--------------------------------------------|
| 0x00  | None     | No signature appended.                     |
| 0x01  | Ed25519  | 64-byte Ed25519 signature of bytes 0..N-64.|

When `signing_method` is `0x01`, the last 64 bytes of the file are the Ed25519 signature computed over all preceding bytes. Public key distribution is out of scope for this specification (typically provided via engine configuration or a separate key file).

---

## Implementation Notes

### Minimum C Implementation

Two files are sufficient:

- `sw3z.h` — struct definitions and API declarations
- `sw3z.c` — reader/writer implementation

Required dependencies:
- `lz4frame.h` / `liblz4` for LZ4 compression
- `zstd.h` / `libzstd` for Zstandard compression (optional, only if zstd assets are used)
- `stdint.h`, `stdio.h`, `stdlib.h`, `string.h`

Optional dependencies:
- An Ed25519 library for archive signing verification (only if signing is used)

No dynamic memory allocation is required for read-only access if the caller provides buffers.

### Recommended C API

```c
// Reader
sw3z_archive_t *sw3z_open(const char *path);
void            sw3z_close(sw3z_archive_t *arc);
int             sw3z_find(sw3z_archive_t *arc, const char *path, sw3z_entry_t *out);
int             sw3z_read(sw3z_archive_t *arc, const sw3z_entry_t *entry,
                          void *buf, uint32_t buf_size);

// Writer (Go tooling side, but C-compatible layout)
sw3z_writer_t  *sw3z_writer_open(const char *path);
int             sw3z_writer_add(sw3z_writer_t *w, const char *path,
                                const void *data, uint32_t size);
int             sw3z_writer_close(sw3z_writer_t *w);
```

### Go Implementation Note

Use `github.com/pierrec/lz4/v4` with `lz4.CompressionLevelOption(lz4.Level9)` for LZ4 offline packing (build/conversion time). For Zstandard, use `github.com/klauspost/compress/zstd`. The LZ4 and Zstandard frame formats are binary-compatible between the Go and C libraries.

### Asset Aliasing

Multiple index entries may reference the same `data_offset` and `compressed_size` to alias one asset under several paths without duplicating data. The writer deduplicates at pack time; the reader requires no special handling — it simply resolves `path_hash → entry → data_offset` as usual.

---

## Constants Reference

```c
#define SW3Z_MAGIC               0x5A335753UL  // "SW3Z" little-endian
#define SW3Z_VERSION             1
#define SW3Z_HEADER_SIZE         24
#define SW3Z_ENTRY_SIZE          40
#define SW3Z_FNV_OFFSET          0xcbf29ce484222325ULL
#define SW3Z_FNV_PRIME           0x00000100000001B3ULL
#define SW3Z_COMPRESS_NONE       0x00
#define SW3Z_COMPRESS_LZ4        0x01
#define SW3Z_COMPRESS_ZSTD       0x02
#define SW3Z_FLAG_EXECUTABLE     0x01
#define SW3Z_FLAG_SYMLINK        0x02
#define SW3Z_FLAG_ALIGNED        0x04
#define SW3Z_ALIGN_MIN           4    // 2^4 = 16 bytes
#define SW3Z_ALIGN_MAX           16   // 2^16 = 64 KiB
#define SW3Z_SIGN_NONE           0x00
#define SW3Z_SIGN_ED25519        0x01
```

---

## Example Archive (Pseudocode)

```
Header:
  magic           = "SW3Z"
  version         = 1
  flags           = 0x0000
  entry_count     = 4
  string_table_size = 74
  data_offset     = 24 + (4 * 40) + 74 = 258

Index[0]:  (regular LZ4 file)
  path_hash       = fnv1a64("maps/q3dm1.bsp")
  string_offset   = 0
  string_length   = 14
  data_offset     = 258
  compressed_size = 48291
  uncompressed_size = 102400
  crc32c          = 0xA1B2C3D4
  compression     = 0x01  // LZ4
  flags           = 0x00
  alignment       = 0x00

Index[1]:  (LZ4 file, page-aligned for GPU upload)
  path_hash       = fnv1a64("textures/base/wall01.png")
  string_offset   = 14
  string_length   = 24
  data_offset     = 49152  // next 4 KiB boundary after 258 + 48291 = 48549
  compressed_size = 8192
  uncompressed_size = 32768
  crc32c          = 0xDEADBEEF
  compression     = 0x01  // LZ4
  flags           = 0x04  // aligned
  alignment       = 12    // 2^12 = 4096

Index[2]:  (directory entry)
  path_hash       = fnv1a64("textures/gothic/")
  string_offset   = 38
  string_length   = 16
  data_offset     = 0
  compressed_size = 0
  uncompressed_size = 0
  crc32c          = 0x00000000
  compression     = 0x00
  flags           = 0x00
  alignment       = 0x00

Index[3]:  (alias — shares data with Index[1])
  path_hash       = fnv1a64("textures/default.png")
  string_offset   = 54
  string_length   = 20
  data_offset     = 49152  // same as Index[1]
  compressed_size = 8192   // same as Index[1]
  uncompressed_size = 32768
  crc32c          = 0xDEADBEEF
  compression     = 0x01
  flags           = 0x00
  alignment       = 0x00

String Table (74 bytes):
  "maps/q3dm1.bsp"              (14 bytes, offset 0)
  "textures/base/wall01.png"    (24 bytes, offset 14)
  "textures/gothic/"            (16 bytes, offset 38)
  "textures/default.png"        (20 bytes, offset 54)

Data:
  [LZ4 frame for q3dm1.bsp]     @ offset 258, 48291 bytes
  [603 bytes zero padding]       @ offset 48549, padding to 49152
  [LZ4 frame for wall01.png]    @ offset 49152, 8192 bytes
```

---

## Security Considerations

Implementations that extract archives to disk SHOULD guard against:

- **Path traversal:** Reject entries with paths containing `..` segments or absolute paths.
- **Symlink targets:** Validate that symlink targets do not escape the extraction directory.
- **Decompression bombs:** Enforce a maximum `uncompressed_size` or ratio limit before allocating buffers.
- **Signature verification:** When `signing_method != 0`, verify the signature before trusting any archive contents.
