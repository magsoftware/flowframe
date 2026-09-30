# ADR-0009: Fixed implementation-wide limits

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead, Security reviewer |
| Phase | P0.6 (values) |
| Related | PRD §13.6, §16.3–16.5; technical-spec §7, §11, §13 |

## Context

Earlier drafts described the renderer timeout and YAML alias limits as "configurable", but no configuration field, CLI option or environment variable existed. Configurable security limits also weaken reproducibility and complicate reviews.

## Alternatives considered

- **Configurable limits in `flowframe.yaml`.** More flexible; adds schema fields, manifest records and review burden.
- **CLI or environment overrides.** Invisible in source control.
- **Fixed limits shipped with each FlowFrame release.** Adopted.

## Decision

1. All implementation-wide limits are fixed constants in the package resource `resources/limits/v1.json`. There is no user-facing override in v0.1.
2. Limit categories: source file bytes; YAML nesting depth and alias expansion; model object counts; theme asset bytes; logo intrinsic dimensions, element count, nesting depth, path-data length and filter regions; embedded icon bytes; final canvas width and height; subprocess stdout and stderr bytes; D2 validate and render timeout.
3. Exceeding a limit is a policy error (exit 5), except the render timeout, which is an execution error (exit 4).
4. Per-asset `maxWidth` and `maxHeight` are layout bounds, not security limits, and cannot raise these limits.
5. Changing a limit requires a FlowFrame release and changelog entry.

## Consequences

- Very large diagrams fail with a clear diagnostic and guidance to split the view.
- Limits are identical locally and in CI.

## Open parameters

| Parameter | Phase | Method |
|---|---|---|
| Numeric values for every category | P0.6 | Measure P0.2 and P0.5 fixtures; set limits at a documented safety margin above the largest legitimate medium fixture. |

## Verification

- P6.3 security suite exercises every limit at the boundary and one unit above it.
