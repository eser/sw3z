# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.9.0] - 2026-03-21

### Added

- Initial SW3Z format specification (v1.0)
- Index-first file layout: 24-byte file header, 40-byte index entries, string table, asset data
- FNV-1a 64-bit path hashing with lowercase/forward-slash normalization
- Per-file compression via `compression` enum (`0x00` = uncompressed, `0x01` = LZ4 Frame, `0x02` = Zstandard)
- Per-file `flags` bitfield (u8): executable (bit 0), symlink (bit 1), aligned (bit 2)
- Data alignment support: configurable power-of-2 alignment (16 B to 64 KiB) for GPU/DMA/NVMe
- Per-asset integrity via CRC32C (Castagnoli) checksum with hardware acceleration (SSE4.2, ARMv8-A)
- LZ4 and Zstandard frame content checksums as second integrity layer
- Ed25519 archive signing via header flags bits 0–1
- Directory entries identified by trailing `/` in path
- Asset aliasing via shared `data_offset` for zero-cost path deduplication
- Stack/override behavior for multi-archive loading with priority ordering
- Flag interaction rules (symlink takes precedence over executable/aligned)
- Forward compatibility: unknown compression values treated as unreadable entries
- Security Considerations section (path traversal, symlink targets, decompression bombs)
- Recommended C API (`sw3z_open`, `sw3z_close`, `sw3z_find`, `sw3z_read`, writer API)
- Go implementation notes using `github.com/pierrec/lz4/v4` and `github.com/klauspost/compress/zstd`
