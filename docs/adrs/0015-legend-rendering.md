# ADR-0015: Legend rendering

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead, Accessibility and design reviewer |
| Phase | P0.4 (proof) |
| Related | PRD §13.5, §13.6; technical-spec §12.3; ADR-0005, ADR-0012 |

## Context

The legend is drawn by the compositor outside D2, but its line samples must look exactly like the relations D2 renders. Legend content was only loosely defined.

## Alternatives considered

- **Draw the legend as a D2 object.** The layout engine would place it; rejected by ADR-0005.
- **Hard-code sample geometry in the compositor.** Drifts from the generator.
- **One shared line-style module used by the generator and the legend.** Adopted.

## Decision

1. `generation/line_styles.py` defines stroke width, dash arrays and arrowhead forms for `synchronous`, `asynchronous`, `not-applicable` and for `directed`, `bidirectional`, `undirected`. The D2 generator and the legend renderer both read it.
2. Architecture and flow legend entries, in this order, include only styles present in the diagram:
   - one entry per interaction present: solid "synchronous", dashed "asynchronous", dotted "not applicable";
   - one entry per directionality present: "directed", "bidirectional", "undirected".
   Samples use `line-dependency` for color so the legend explains patterns and arrowheads, not colors.
3. Sequence legend (only when `display.legend: true`) includes only what is shown: "message" (solid, single arrowhead), "note", and "step number" when numbering is on.
4. Legend text uses `text-secondary`; wrapping follows ADR-0007.
5. An empty legend occupies no band.

## Consequences

- Changing a line style changes both D2 and legend in one place.

## Verification

- Unit tests: legend entries equal the set of styles in the IR, in the defined order.
- Snapshot comparison of legend samples and rendered relations in P4.8.
