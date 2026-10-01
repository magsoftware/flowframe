# ADR-0031: FlowFrame SVG writer with pluggable geometry providers

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-10-01 |
| Owner | Tech lead, Product owner, Accessibility and design reviewer, Security reviewer |
| Phase | P0.1 (question), P0.2–P0.4 (spike) |
| Related | PRD §5, §6, §11, §12, §13.6–§13.9; technical-spec §4, §10, §11, §12, §14; ADR-0002, ADR-0003, ADR-0005, ADR-0006, ADR-0007, ADR-0008, ADR-0012, ADR-0014, ADR-0015, ADR-0024, ADR-0026, ADR-0029 |

## Context

FlowFrame already owns almost everything around D2. It decides the semantics, the shapes and token colors (ADR-0012), the label text and wrapping (ADR-0011), the decorations (ADR-0005), the legend content (ADR-0015), the accessibility metadata (ADR-0014) and the canonical form of the published SVG (ADR-0008). D2 contributes two things: it runs the ELK layout and it draws the laid-out diagram as SVG.

Because the drawing happens inside D2, a large part of the v0.1 design exists to work with an SVG that FlowFrame did not write:

- recovering a mapping from rendered SVG groups back to source IDs before accessibility metadata can be added (ADR-0014),
- inventorying nondeterministic fields and renderer CSS and normalizing them away (ADR-0008),
- keeping a second text measurement in step with D2 within one pixel, so that composed decorations line up with the body (ADR-0007, ADR-0005),
- drawing legend samples with a separate renderer that must match D2 edges (ADR-0015),
- pinning, installing and verifying a per-platform executable and guarding the process boundary (ADR-0002, ADR-0006),
- generating, escaping and linting D2 text that exists only to be parsed again (technical-spec §10).

The D2 v0.8.2 release notes show the cost of this dependency. That release replaced the embedded ELK.js with a native Go port and warns that layouts change as a result, so existing snapshots have to be regenerated. Every D2 upgrade is therefore a reviewed update of all goldens, driven by internals FlowFrame does not control.

PRD §6.1 already requires that "FlowFrame schemas MUST remain independent enough to allow another renderer in the future". This ADR asks whether the renderer should be FlowFrame from the start, with the layout engine reduced to a function that returns geometry.

## Alternatives considered

- **Keep D2 as the renderer (current baseline).** Mature shapes and visual polish without design work; TALA is reachable later through the same tool. Costs everything listed under Context.
- **Keep D2 as the renderer, place decorations natively in D2 (ADR-0026).** Removes the compositor, but keeps the hook mapping, the normalization of foreign SVG, the binary pinning and the upgrade churn.
- **D2 for layout only.** FlowFrame consumes the laid-out diagram (`d2target.Diagram` from the Go library, or the result of `compile` in the `@d2lang/d2` package) and writes SVG itself. Keeps D2 layout behavior and a path to TALA, but still pins D2, still generates D2 text and depends on an internal data structure that has no stability promise.
- **ELK as a geometry provider, FlowFrame writes SVG.** Proposed here. FlowFrame sends ELK a graph with measured node and label sizes and receives positions and edge routes.
- **Graphviz `dot` as the geometry provider.** Stable and widely packaged, with clusters for nesting. Rejected as the default because it is a native dependency again, and its handling of nested clusters and edge labels is weaker than ELK layered for the architecture family. It can be added later as another provider.

## Proposed decision

Adopted only if the spike under **Verification** passes. Otherwise this ADR is Rejected and the D2 baseline stays, with or without ADR-0026.

1. **Rendering pipeline.** Family IR → layout request → geometry provider → geometry → FlowFrame SVG writer → canonical `diagram.svg`. Validation, selection, projection, the manifest and publication do not change.
2. **Layout request and geometry are FlowFrame contracts.** The layout request is a JSON document carrying nodes with measured sizes, container hierarchy, edges, label sizes, direction and the profile. The geometry is a JSON document carrying positions, sizes, edge routes as polylines and label anchors, all keyed by source ID. Both are canonical JSON and are the golden-test boundary that `diagram.d2` is today.
3. **Text is measured once.** The ADR-0007 measurement module produces every label and decoration size before layout. The writer uses the same module, so there is no parity requirement with an external measurement.
4. **Architecture and flow use ELK layered.** The provider runs `elkjs` as a packaged, hash-pinned JavaScript resource. The spike chooses between an embedded JavaScript engine (no external runtime) and Node as a subprocess. The ADR-0003 open parameters become ELK option sets for `hierarchical` and `compact` and a direction mapping.
5. **Sequence diagrams use a FlowFrame layout.** Participants are columns spaced by measured width, each message and note takes one row in IR order, and a self-message is a loop on its own lifeline. The layout is a pure function of the Sequence IR (PRD §11.3) and needs no layout engine.
6. **The writer draws everything.** It draws the ADR-0012 shapes from FlowFrame-owned geometry, edges with the ADR-0015 line styles and arrowheads, containers, icons as one `<symbol>` per unique asset referenced by `<use>`, decorations and the legend. It emits the ADR-0014 accessibility metadata on the elements it writes, and it writes canonical output directly: fixed attribute order, fixed number precision and no generated IDs that depend on the run.
7. **Geometry providers are pluggable.** The provider interface is internal in v0.1. A D2-backed provider, and through it TALA, may be added later behind the same interface, which keeps PRD §6.2 intact as a post-MVP option.

## Consequences

If accepted:

- **Removed from v0.1:** the D2 executable with its pinning, installer and `FLOWFRAME_D2` override (ADR-0006), the D2 process boundary (ADR-0002 item 5, technical-spec §11), D2 generation and its header (technical-spec §10), normalization of foreign SVG (most of ADR-0008), the group-to-source mapping proof (ADR-0014 hooks), the compositor that measures and shifts a rendered body (ADR-0005, technical-spec §12.3), the second legend renderer (ADR-0015) and text-measurement parity with D2 (ADR-0007). ADR-0026 becomes moot.
- **Added:** the SVG writer for twelve distinct shapes, edges, arrowheads, line patterns and containers, the sequence layout, ELK option tuning, and font embedding or subsetting per ADR-0007. Visual polish that D2 gives for free has to be designed and reviewed.
- **Lost for now:** TALA and the `diagram.d2` artifact. `diagram.d2` is not meant for manual editing (PRD §5), and its role as an inspectable intermediate moves to the layout request and geometry JSON.
- **Licensing.** `elkjs` is EPL-2.0 and joins the ADR-0024 inventory. Shape geometry is drawn by FlowFrame, not copied from D2 (MPL-2.0).
- **Other proposed ADRs.** ADR-0027 still applies: embedding the logo as a separate image document avoids ID rewriting in any writer. ADR-0028 and ADR-0030 are unaffected. Under ADR-0029, `compile` would emit the layout request instead of D2.
- **Documents to change in the same change that accepts this ADR:** PRD §5, §6, §12, §16 and the artifact trees and manifest example; technical-spec §3, §4, §5, §10, §11, §12.3, §14, §15; ADR-0002, ADR-0003, ADR-0005, ADR-0006, ADR-0008, ADR-0014 and ADR-0015 superseded or amended; implementation-plan P0.2–P0.4 replaced by the spike and the later phases re-sequenced around the writer.

## Verification

A timeboxed spike of three working days, run before the P0.2 D2 work, on `examples/payments/infrastructure-view.yaml` and one sequence fixture with self-messages and notes. It produces the same view with the current baseline (D2 + ELK) and with this proposal, and passes only if all of the following hold:

- **Layout quality.** The accessibility and design reviewer judges nested containers, edge routing and label placement at least as readable as the D2 baseline, without manual coordinates.
- **Determinism.** Two runs in two directories produce byte-identical geometry JSON and SVG, on Linux and on macOS, and both platforms produce identical bytes. Coordinates are rounded by the writer to a fixed precision; the spike records whether any rounding boundary flips between platforms.
- **Runtime.** The chosen way to run `elkjs` installs from wheels on every supported platform of technical-spec §3.1 without a compiler, or, if Node is required, the dependency is acceptable to the product owner. Build time for the medium fixture stays within the draft ADR-0023 budget.
- **Accessibility and consumers.** The SVG carries the ADR-0014 metadata for every semantic object and renders correctly in every ADR-0022 consumer.
- **Effort estimate.** The spike records an estimate for the full writer, covering all shapes, line styles, icons, decorations and legend, next to the estimate of the D2 baseline work it replaces (P0.2–P0.4, ADR-0008 normalization, ADR-0014 hooks, compositor).

The result, with screenshots of both renderings, is recorded here. A failed criterion either rejects this ADR or narrows it to a recorded variant, such as D2 for layout only.
