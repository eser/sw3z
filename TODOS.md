# TODOs

## Machine-readable format definition

Create a Kaitai Struct (`.ksy`) or similar machine-readable description of the SW3Z format. This enables auto-generated parsers in any language and serves as an unambiguous test oracle for verifying implementations.

- **Effort:** M
- **Priority:** P2
- **Blocked by:** Spec v1.0 finalization
- **Context:** The prose spec in README.md is the source of truth, but a machine-readable schema would eliminate ambiguity in field sizes, offsets, and enum values. Kaitai Struct can generate parsers for C++, Go, Python, Java, and more from a single `.ksy` file.
