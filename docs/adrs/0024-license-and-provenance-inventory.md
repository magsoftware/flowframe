# ADR-0024: License and provenance inventory

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Product owner, Tech lead |
| Phase | P0.6 (decision), P8.1 (SBOM) |
| Related | PRD §13.3, §13.8; implementation-plan P8.1, P8.2; ADR-0013 |

## Context

Theme packs, the built-in font set and the icon pack carry third-party assets. A machine-readable SPDX inventory would allow automatic coverage checks but extends the MVP contract.

## Alternatives considered

- **Machine-readable SPDX inventory per pack.** Automatic checks; additional schema and tooling. Deferred to post-MVP.
- **Human-readable `LICENSES.md` with a manual release review.** Adopted for v0.1.

## Decision

1. Every Theme/Brand Pack, the built-in font set and the generic icon pack include a UTF-8 `LICENSES.md`. For each file it lists the path, license name and source.
2. v0.1 checks automatically only that the file exists and is referenced. License coverage is reviewed manually as part of the release checklist (P8.4).
3. The release produces a CycloneDX JSON SBOM for Python dependencies and bundled resources (P8.1).
4. A machine-readable SPDX inventory is post-MVP.

## Consequences

- License errors in custom themes are the project maintainer's responsibility; FlowFrame does not verify content.

## Verification

- Schema and loader tests: missing `licenses` reference or missing file is an error.
- P8.4 checklist item: manual license review signed off.
