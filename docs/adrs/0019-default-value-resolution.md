# ADR-0019: Single source of default values

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead |
| Phase | P1 (mechanism), P0.6 (verification of PRD defaults) |
| Related | PRD §10.8, §13.2; technical-spec §9.1 |

## Context

The plan asked for display and layout defaults to be "encoded" in JSON Schema, while the technical specification required one shared resolver. Defaults that depend on `detail` and family cannot be expressed with JSON Schema `default`, and `jsonschema` does not apply defaults anyway.

## Decision

1. Defaults live in the package data file `resources/defaults/v1.json`, covering the built-in configuration (PRD §13.2), View display and layout defaults per family and detail (PRD §10.8), integration-flow relation-semantic defaults and theme numeric decoration defaults.
2. One resolver module (`domain/defaults.py`) applies them after schema validation. CLI, projectors and generators never hard-code defaults.
3. JSON Schema `default` annotations are informative only. A contract test asserts they agree with the data file wherever both exist.

## Consequences

- Changing a default is one data change plus golden updates.

## Open parameters

| Parameter | Phase | Method |
|---|---|---|
| Confirmation of PRD §10.8 defaults on representative output | P0.6 | Render the P0 corpus with defaults; any change updates the PRD first. |

## Verification

- Contract test for schema annotations versus `defaults/v1.json`.
- Resolver unit tests for every family and detail combination.
