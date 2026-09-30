# ADR-0022: Supported SVG consumers

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-09-30 |
| Owner | Product owner, Accessibility and design reviewer |
| Phase | P0.1 (before P0.3 and P0.4 experiments) |
| Related | PRD §16.3; technical-spec §12.3, §14; ADR-0007, ADR-0014 |

## Context

Text rendering, embedded fonts, logos and accessibility metadata behave differently across browsers and documentation platforms. P0 experiments and P6.4 checks need a fixed target list.

## Alternatives considered

Candidate list:

- browsers: current stable Chromium, Firefox and Safari;
- documentation platforms: GitHub and GitLab Markdown image rendering, MkDocs Material;
- embedding modes: `<img>` and inline SVG.

## Decision

To be accepted in P0.1: the consumer list with versions and embedding modes. Each entry states which properties are verified (layout, fonts, logo, accessibility tree).

## Consequences

- Consumers outside the list are best effort.

## Verification

- P0.3, P0.4 and P6.4 run their checks against every listed consumer.
