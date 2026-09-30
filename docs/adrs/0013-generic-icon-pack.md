# ADR-0013: Generic icon pack

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead, Security reviewer |
| Phase | P0.3 (embedding proof), P0.6 (icon set) |
| Related | PRD §13.8, §16.1; technical-spec §10, §13.2, §15; ADR-0012, ADR-0024 |

## Context

The PRD required "approved, sanitized icons" embedded in D2, and recorded `assetProcessing` "when vector logos/icons have been sanitized". It was unclear which sanitizer profile applies to icons, whether sanitization happens at runtime, and how logo and icon records differ in the manifest.

## Alternatives considered

- **Sanitize icons at every build.** Repeats deterministic work and makes every architecture manifest carry sanitizer records for framework resources.
- **Pre-sanitize at package build and verify hashes at runtime.** Adopted.

## Decision

1. The generic pack lives in `resources/icons/generic/` with a versioned `index.json` mapping every element kind to an icon file or explicit `null`, plus SHA-256 and license reference per file.
2. Icon files are sanitized with the `flowframe-svg-logo/v1` profile when the package is built. The committed files are the canonical sanitized bytes.
3. At runtime FlowFrame verifies each used icon's SHA-256 against the index. A mismatch, missing entry or missing file is an invalid-pack error.
4. Icons are recorded in the manifest `resources` array (`kind: icon`). `assetProcessing` records only project Theme/Brand Pack assets sanitized at build time (logos), and each entry carries `kind: logo`.
5. The D2 generator embeds each unique icon once as a bounded data URI declaration reused through classes. The D2 lint accepts only data URIs whose decoded bytes match an index hash.
6. Mapping is by semantic kind only; free-text `technology` is never used to choose an icon.

## Consequences

- Architecture builds without a logo have no `assetProcessing` section.
- Icon changes are package changes with reviewed golden updates.

## Open parameters

| Parameter | Phase | Method |
|---|---|---|
| Icon set and license | P0.6 | Choose a permissively licensed generic set covering all 13 kinds or set explicit `null`. |
| Proof that one declaration per icon is reused offline by the pinned D2 | P0.3 | Render fixtures with several nodes and classes sharing one icon. |

## Verification

- Package build test: every icon passes the sanitizer and its hash matches the index.
- Runtime tests: tampered icon bytes, missing entry and missing file fail.
