# ADR-0004: Schema authority and contract versioning

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead, Product owner |
| Phase | P0.1 |
| Related | PRD §6.3, §16.1; technical-spec §6.1, §21 |

## Context

FlowFrame has seven public machine-readable contracts: Project Configuration, System Model, View Specification, Theme/Brand Pack, manifest, diagnostics output and the D2 pin resource. All schemas are closed (`additionalProperties: false`). Post-MVP work adds subtypes and fields, so the versioning rules must be clear before v1 schemas are frozen.

## Alternatives considered

- **Python model classes as the contract.** Rejected: not usable by editors, AI adapters or other languages.
- **Open schemas that ignore unknown fields.** Rejected: misspelled properties would be silently ignored.
- **Semantic versions with additive minor versions (`v1.1`).** Considered. Rejected for v0.x because a closed contract with an additive minor still rejects the new field in an older reader, so compatibility gains are small and the rules become harder to explain.
- **Closed schemas, one version identifier per accepted language, bounded multi-version support.** Adopted.

## Decision

1. Published JSON Schema (Draft 2020-12) files are the authoritative structural contracts. Python classes are internal.
2. Contract identifiers have the form `flowframe-<contract>/vN`. Any change to the accepted language (new field, new enum value, changed meaning, removed field, newly required field) requires a new `N`.
3. A FlowFrame release reads the current and the immediately previous version of each source contract (configuration, model, view, theme). v0.1 reads only v1.
4. The publication ownership check accepts every manifest version the running release can read (ADR-0010).
5. An unsupported version produces one stable diagnostic with migration guidance. Automatic migration tooling is post-MVP.
6. Each schema has a stable `$id` of the form `https://flowframe.dev/schemas/<contract>/vN.schema.json`. The URL is an identifier only; schemas are always resolved from package resources, never fetched.
7. Post-MVP subtypes (for example `architecture/deployment`) require `flowframe-view/v2`.

## Consequences

- Misspellings fail fast.
- Each post-MVP enum addition is a visible contract change with release notes.
- Maintaining two readable versions per contract costs test fixtures from v0.2 onward.

## Verification

- P1.2: every schema resolves offline; duplicate `$id` fails a test; unsupported `schemaVersion` yields one stable diagnostic.
