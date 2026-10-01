# ADR-0026: Decorations and legend placed by D2

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-10-01 |
| Owner | Tech lead, Accessibility and design reviewer |
| Phase | P0.1 (question), P0.3/P0.4 (proof) |
| Related | PRD §13.6, §13.7; technical-spec §12.3; ADR-0005, ADR-0007, ADR-0014, ADR-0015 |

## Context

ADR-0005 rejects emitting decorations as D2 objects because "the layout engine would move them" and builds a FlowFrame compositor instead. That compositor carries a large share of the v0.1 design:

- the band and width algorithm of PRD §13.6 and technical-spec §12.3,
- deterministic text measurement for decorations that must agree with D2 body text (ADR-0007),
- a second renderer for relation line samples kept in sync with D2 output (ADR-0015),
- logo measurement, scaling and inlining with collision-safe ID rewriting (technical-spec §12.3).

The rejected alternative was evaluated only as "decorations as ordinary graph objects". Two D2 features that address the stated reason were not considered:

- **`near` with a constant position** (`top-left`, `top-center`, `top-right`, `center-left`, `center-right`, `bottom-left`, `bottom-center`, `bottom-right`). The D2 documentation lists it as supported by every layout engine; only `near` pointing at another object is TALA-specific. Constant-near objects are placed outside the laid-out graph rather than inside it, so ELK does not move them among the semantic nodes.
- **`vars.d2-legend`**, a native legend rendered from a small diagram of sample shapes and connections. Objects with `opacity: 0` are left out, so a legend can show connections only.

If these features cover the v0.1 decoration contract, most of the compositor becomes unnecessary.

## Alternatives considered

- **Keep ADR-0005 (FlowFrame compositor).** Full control over bands, gaps and slot geometry. Costs the work listed above, including text-measurement parity with D2.
- **Title, subtitle, footer and logo as constant-near D2 objects; legend through `d2-legend`; FlowFrame post-processing only for accessibility metadata.** Proposed here, subject to the P0 proof below.
- **Hybrid: text decorations in D2, logo or legend in the compositor.** Considered only if the proof fails for one decoration kind. ADR-0007 already requires that a hybrid has a defined boundary, so a hybrid would be recorded explicitly, never reached by accident.

## Proposed decision

Adopted only if every item under **Verification** passes on the pinned D2. Otherwise this ADR is Rejected and ADR-0005 remains in force.

1. The D2 generator emits the title block, the footer and the logo as root-level objects with constant `near` positions taken from Project Configuration. The slot names in PRD §13.2 map one to one onto D2 constants.
2. The title block is one object containing the title and, when present, the subtitle. The footer is one plain-text object produced from the footer template. Both use the label wrapping of ADR-0011 with `decoration-text-max-width`. Breaks are inserted before D2 sees the text, so no compositor measurement is needed.
3. The logo is a constant-near object whose image is the sanitized asset embedded as a data URI. Its size is the "contain" box from the asset's `maxWidth` and `maxHeight`, computed from the sanitized intrinsic view box. ADR-0027 defines the embedding.
4. The legend is emitted as `vars.d2-legend`. The legend content of ADR-0015 stays as it is: entries, their order and the rule of listing only styles present in the diagram. Samples reuse the classes that style real relations, so `generation/line_styles.py` stays the only definition of line styles, and there is no second renderer.
5. Slot validation (PRD §13.2, §13.6) is unchanged: one primary decoration per slot, evaluated after defaults.
6. FlowFrame post-processing of the rendered SVG is limited to accessibility metadata (ADR-0014) and canonicalization (ADR-0008). It does not move, measure or resize geometry.

## Consequences

If accepted:

- Removed from v0.1: the band and width algorithm, the decoration text-fragment renderer, legend sample drawing, logo inlining with ID rewriting, and the requirement that compositor measurement match D2 body label widths within one pixel. Label wrapping still needs a measurement module, but only to choose break points, not to position geometry.
- `decoration-gap` and `decoration-padding-x` are kept only if the pinned D2 exposes an equivalent spacing control. Otherwise they are removed from the theme contract before v1 is published.
- Decoration placement depends on the pinned D2 version, like the body layout. D2 pin upgrades already go through reviewed golden updates (ADR-0006, ADR-0021).
- Documents to change in the same change that accepts this ADR: PRD §7, §13.6, §13.7, §22 (AC 9); technical-spec §4, §5, §12.3; ADR-0005 and ADR-0015 marked superseded; ADR-0007 decision 6 and open parameters narrowed; implementation-plan P0.4 and P4.7–P4.10 reduced.

## Verification

P0.3/P0.4 fixtures for all three MVP families on the pinned D2 with ELK:

- every title, footer and logo slot of PRD §13.2, alone and in every valid combination, never overlaps body objects or other decorations,
- a long wrapped title and a decoration wider than the body,
- constant-near objects together with a sequence diagram body, either at the root or with the sequence wrapped in a container,
- `d2-legend` with every interaction and directionality sample, using the same classes as relations,
- a data-URI logo rendered offline in every ADR-0022 consumer,
- repeat renders in two directories produce identical canonical SVG,
- the ADR-0014 hook mechanism still locates every semantic object and the decorations.

Any failure is recorded here and the ADR is Rejected or narrowed to a recorded hybrid.
