# ADR-0012: Element shapes and theme token application

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead, Accessibility and design reviewer |
| Phase | P0.3 (rendering check), P0.6 (acceptance) |
| Related | PRD §13.3–13.5, §13.7, §13.9, §22; technical-spec §10; ADR-0013 |

## Context

The PRD referred to a "semantic shape" without defining it, and did not say which token colors which part of a diagram. Without that, contrast cannot be checked deterministically and grayscale readability cannot be assessed. The tokens `primary` and `secondary` were required but never used.

## Alternatives considered

- **Theme-defined shapes.** Rejected: shapes carry meaning and are framework-owned.
- **One rectangle for every kind, distinguished by icon only.** Fails when an icon is intentionally `null`.
- **Distinct framework shapes plus a fixed token application matrix.** Adopted.

## Decision

**Shapes** (D2 `shape` values):

| Kind | Shape | Additional non-color cue |
|---|---|---|
| actor | person | — |
| external-system | rectangle | dashed border |
| web-application | page | — |
| service | rectangle | — |
| worker | step | — |
| gateway | hexagon | — |
| database | cylinder | — |
| cache | stored_data | — |
| queue | queue | — |
| storage | package | — |
| identity-provider | oval | — |
| secret-store | diamond | — |
| security-control | parallelogram | — |

Boundaries are rectangular containers. `trust-zone` uses a dashed border; other boundary kinds use a solid border. The boundary kind also appears as label text (ADR-0011).

**Token application**:

| Object | Fill | Stroke | Text |
|---|---|---|---|
| Canvas | `background` | — | — |
| Element | mapped element token | `border` | `text-primary` |
| Boundary | mapped boundary token | `border` | `text-primary` |
| System boundary | `surface` | `border` | `text-primary` |
| Relation / message | — | mapped relation token (sequence messages: `line-request`) | `text-primary` |
| Sequence note | `surface-muted` | `border` | `text-primary` |
| Title / subtitle / footer | — | — | `title-text` / `subtitle-text` / `footer-text` |
| Legend | — | relation tokens for samples | `text-secondary` |

`primary` and `secondary` are removed from the required token list.

**Contrast pairs checked by the theme validator (FFT errors)**:

- Text 4.5:1: `text-primary` against every element fill token, every boundary fill token, `surface`, `surface-muted` and `background`; `title-text`, `subtitle-text`, `footer-text` and `text-secondary` against `background`.
- Non-text 3:1: every `line-*` token and `border` against `background`, `surface`, `surface-muted`, `network` and `security`.

Pairs are computed from the resolved mappings, so a custom mapping adds its tokens to the same checks.

## Consequences

- Grayscale distinction of element kinds does not depend on icons or color.
- Theme schema and PRD token list drop `primary` and `secondary`.

## Open parameters

| Parameter | Phase | Method |
|---|---|---|
| Confirmation that every shape renders correctly with ELK inside nested containers and as a sequence participant | P0.3 | Render all kinds in the P0 fixtures; replace a shape only through an amendment to this ADR. |

## Verification

- Contrast validator unit tests for the built-in and custom themes.
- Grayscale snapshot review in P6.4.
