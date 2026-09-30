# ADR-0021: Golden and snapshot policy

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead, Accessibility and design reviewer |
| Phase | P0.6 (open parameters) |
| Related | PRD §16.1, §19.1, §23; technical-spec §19; ADR-0008 |

## Context

D2 goldens are platform-independent Python output. SVG output depends on the per-platform D2 executable. Consumers may commit generated SVG; line-ending conversion or formatters would then modify artifacts and block the next build (ADR-0010 ownership check).

## Decision

1. D2 goldens are one committed set shared by all platforms.
2. SVG snapshots are the canonical published bytes (ADR-0008).
3. Every snapshot change requires Accessibility and design reviewer approval; a snapshot update alone is not evidence of acceptability.
4. Consumer documentation recommends `.gitattributes` entries `*.svg -text`, `*.d2 -text` and `manifest.json -text` for output directories, and excluding them from formatters and pre-commit hooks.

## Consequences

- Consumers that ignore the recommendation may see "modified artifacts" rejections; the diagnostic hint points to this guidance.

## Open parameters

| Parameter | Phase | Method |
|---|---|---|
| One SVG snapshot set or one per platform | P0.6 | Compare canonical SVG from the pinned D2 on macOS and Linux for all fixtures. |
| Recommended consumer policy: commit SVG or generate in CI | P0.6 | Evaluate review diffs and CI cost on the example project. |
| Rasterizer for grayscale and visual review | P0.6 | Pin one offline rasterizer (candidate: `resvg`) or document browser-based manual review. |

## Verification

- CI fails when a snapshot differs and no approved update is included.
