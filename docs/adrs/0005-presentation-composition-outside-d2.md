# ADR-0005: Global presentation composed outside D2 layout

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead |
| Phase | P0.1 (decision), P0.4 (proof) |
| Related | PRD §13.6, §13.7; technical-spec §12; ADR-0007, ADR-0014, ADR-0015 |

## Context

Title, subtitle, footer, logo, legend and accessibility summary must look identical across diagram families and must never overlap the diagram body. Layout engines position objects based on the graph and do not guarantee reserved regions.

## Alternatives considered

- **Emit decorations as D2 objects.** The layout engine would move them, sequence diagrams have different constraints, and placement would change between engines.
- **Post-process with a browser or headless renderer.** Host-dependent and not reproducible offline.
- **Deterministic SVG compositor after D2 renders the body.** Adopted.

## Decision

1. D2 renders only the semantic body.
2. A FlowFrame compositor measures the body, renders decorations and the legend as independent SVG fragments, expands the canvas with bands (PRD §13.6) and translates the body without changing its internal geometry.
3. The compositor also injects accessibility metadata (ADR-0014).
4. The layout engine is never asked to position decorations.

## Consequences

- Decorations are engine-independent and deterministic.
- The compositor needs deterministic text measurement (ADR-0007) and must replicate relation line styles for the legend (ADR-0015).
- `render` (body-only) output contains no decorations, legend or accessibility summary.

## Verification

- P0.4: the band algorithm never clips or overlays the body on SVG emitted for every MVP family.
- P4 compositor tests cover all valid slot combinations and canvas limits.
