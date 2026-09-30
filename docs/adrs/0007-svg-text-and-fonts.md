# ADR-0007: SVG text representation, fonts and measurement

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead, Accessibility and design reviewer |
| Phase | P0.1 (decisions 1–6), P0.4 (open parameters) |
| Related | PRD §13.3, §13.6, §13.7, §16.3; technical-spec §12.3; ADR-0005, ADR-0011, ADR-0022 |

## Context

The compositor wraps and measures decoration text; D2 measures body text. Both must use the same pinned fonts. SVG consumers rasterize `<text>` differently, while text converted to paths loses native selection and search. No default font family has been chosen yet, and a single font cannot cover every Unicode script.

## Alternatives considered

| Option | Portability | Accessibility | Output size |
|---|---|---|---|
| `<text>` with fonts embedded by D2 and the compositor | consumer rasterization differs slightly | native text, searchable | medium |
| All visible text converted to paths | identical glyph shapes everywhere | requires per-object accessible names and a textual summary | large |
| Hybrid (body `<text>`, decorations paths) | mixed | mixed | medium |

Default font candidates (all SIL OFL 1.1): Noto Sans, IBM Plex Sans, Inter.

Measurement candidates: `fontTools` advance widths without shaping; `uharfbuzz` shaping with kerning and ligatures.

## Decision

Binding decisions:

1. Custom fonts are TTF only, with exactly four faces: regular, bold, italic, semibold. No host font discovery or fallback.
2. The same pinned font bytes are passed to D2 (`--font-*` flags) and used by the compositor.
3. Grapheme boundaries are computed with the `regex` package (`\X`).
4. **Glyph coverage is a documented limitation in v0.1**: the documentation lists the scripts covered by the built-in font set. Text outside that coverage renders with missing-glyph boxes. There is no glyph-coverage validator in v0.1; it is a post-MVP candidate.
5. Font sizes are framework-owned.

6. Exactly one output-wide text policy applies to D2 body text and FlowFrame decorations. Path conversion may be selected only if ADR-0014 accessibility requirements remain satisfied; a hybrid policy requires a defined boundary and tests of both portability and accessibility.

The text policy, the built-in `flowframe-default` font family with its coverage statement, and the measurement library are open parameters filled in P0.4. The measurement library must match D2 body label widths within one pixel per line on the P0 fixtures; shaping is required only if that bound cannot be met without it.

## Consequences

- Label wrapping (ADR-0011) and decoration wrapping share one measurement module.
- Non-Latin labels may render with missing glyphs; users are told which scripts are covered.

## Open parameters

| Parameter | Phase | Method |
|---|---|---|
| Text policy (one of the alternatives above) | P0.4 | Render the P0 fixtures with each option in every ADR-0022 consumer; compare appearance, size and accessibility tree. |
| Default font family and documented script coverage | P0.4 | Compare coverage, metrics stability, license (SIL OFL 1.1 candidates) and D2 compatibility. |
| Measurement library and need for shaping | P0.4 | Compare compositor measurements with D2 body label widths on the fixture set. |

## Verification

- P0.4 acceptance in the implementation plan.
- P4.7: wrapping tests with Latin, accented, CJK and emoji text; the result for uncovered text matches the documented limitation.
