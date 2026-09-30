# ADR-0016: Diagnostics contract

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead |
| Phase | P1 (before P2.4) |
| Related | PRD §15.3, §17, §17.1; technical-spec §8.2; ADR-0025 |

## Context

JSON diagnostics are consumed by CI and by the AI repair loop, but had no schema. Paths, ordering, stdout behavior in JSON mode and several category assignments were undefined.

## Alternatives considered

- **Document fields informally.** Consumers break on wording or shape changes.
- **Minimal versioned JSON Schema.** Adopted.

## Decision

1. Schema `flowframe-diagnostics/v1` (`resources/schemas/flowframe-diagnostics.schema.json`) describes the JSON array written to stderr. Each entry has `code`, `severity`, `category`, `message`, optional `location` (`file`, `line`, `column`, `pointer`), optional `related` (array of locations), optional `hint` and optional `debug` (object, only with `--debug`).
2. `location.file` is relative to the resolved project root. For `validate --theme`, it is relative to the theme directory. Absolute paths never appear outside `debug`.
3. Sort order: file, line, column, code, message; diagnostics without a location come last, sorted by code and message.
4. In JSON mode stdout is empty. In text mode, artifact paths may be printed to stdout.
5. Code prefixes: FFC, FFS, FFM, FFV, FFT, FFI, FFD, FFR, FFX, plus **FFA** for AI adapter and source-conformance findings.
6. Category assignments that were previously ambiguous:

   | Case | Category | Exit |
   |---|---|---:|
   | `--layout` value unsupported on the CLI | usage | 2 |
   | `render.layoutEngine` unsupported in the configuration | validation | 1 |
   | Source file (model, view) outside the project root without symlinks | validation | 1 |
   | Symlink or theme asset escaping the project root | policy | 5 |
   | Missing or malformed FlowFrame D2 header, or theme fingerprint mismatch in `render` | validation | 1 |
   | Refused overwrite of a non-artifact file or forbidden output directory | policy | 5 |
   | Missing optional AI adapter | dependency | 3 |
   | Source-conformance discrepancy | FFA warning | 0 |

7. `docs/diagnostics.md` is the code registry. Codes that exist only in prereleases (for example alpha.1 rejection of unsupported presentation) are marked `prerelease-only` and removed before v0.1.

## Consequences

- The AI adapter and CI parse one documented structure.
- Adding a field is a new diagnostics schema version (ADR-0004).

## Verification

- Every CLI contract test in JSON mode validates stderr against the schema and asserts empty stdout.
- A registry test fails when a code used in the source is missing from `docs/diagnostics.md`.
