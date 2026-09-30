# ADR-0020: Theme aggregate hash algorithm

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead |
| Phase | P1 |
| Related | PRD §16.1, §17; technical-spec §6.3, §10 |

## Context

The theme aggregate hash appears in the manifest (`theme.sha256`) and in the generated D2 header, where `render` compares it with the resolved theme. The domain tag, length encoding and set of covered files were unspecified, and the PRD said "referenced assets" in one place and "declared" in another.

## Decision

1. Covered entries: `theme.yaml`, the license inventory, **every declared asset** in `assets`, and every custom font file in `typography.fonts`. For a theme using `fontSet`, each font file of the set is an entry named `resource:fonts/<set-id>/<file>`.
2. Entry names are normalized POSIX paths relative to the theme directory (or the resource names above), UTF-8 encoded, sorted bytewise.
3. Hash input: the ASCII bytes `flowframe-theme-hash/v1\n`, then for each entry: 4-byte big-endian name length, name bytes, 8-byte big-endian content length, raw content bytes.
4. The result is lowercase hexadecimal SHA-256.
5. Built-in themes use the same algorithm with resource names.

## Consequences

- A declared but unused asset changes the hash: the hash identifies the pack, not its use.

## Verification

- P1.9 adds a committed test vector (a small fixture theme and its expected hash).
- Header and manifest values are compared in P3 integration tests.
