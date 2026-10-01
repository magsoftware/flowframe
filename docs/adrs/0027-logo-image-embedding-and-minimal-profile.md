# ADR-0027: Logo embedded as an image with a minimal v1 sanitizer profile

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-10-01 |
| Owner | Tech lead, Security reviewer |
| Phase | P0.1 (question), P0.5 (proof) |
| Related | PRD §13.6, §16.4, §16.5; technical-spec §3, §12.3, §13.2; ADR-0013, ADR-0014, ADR-0026 |

## Context

The `flowframe-svg-logo/v1` profile (PRD §16.5) must support gradients, clip paths, masks, internal `use`, ten filter primitives and `<style>` validated by a real CSS parser. Technical-spec §12.3 inlines the sanitized logo nodes into the final SVG, which requires deterministic renaming of colliding IDs and rewriting of every fragment reference. Together they make the sanitizer the largest security-sensitive component of v0.1. They also add `tinycss2` as a dependency and make the P0.5 spike long.

Two properties of the problem allow a smaller design:

- An SVG document referenced through an `<image>` element is rendered by browsers as a separate static document: scripts do not run and external resources are not loaded. Its IDs live in their own document, so they cannot collide with the body.
- Logos are supplied by project maintainers once per project, not by arbitrary authors per diagram. Asking for a simple export (presentation attributes, no filters) is a reasonable one-time cost.

## Alternatives considered

- **Keep the full profile and inline logo nodes.** Rich logos are accepted as is. Costs the CSS parser, the filter and mask allowlists, ID rewriting and the larger fixture corpus.
- **Embed the sanitized logo as a separate image document, and start v1 with a minimal profile.** Proposed here.
- **Accept only raster logos (PNG).** Simplest to embed, but logos lose sharpness when scaled, and embedded rasters are rejected elsewhere in the profile.

## Proposed decision

1. The sanitized logo is embedded as a separate image document: the `image/svg+xml` data URI of a D2 image object (ADR-0026), or an SVG `<image>` element if ADR-0026 is rejected. Logo nodes are never inlined into the diagram document, so no ID rewriting or reference renaming is needed.
2. `flowframe-svg-logo/v1` accepts only:
   - structure and shapes: `svg`, `g`, `defs`, `title`, `desc`, `path`, `rect`, `circle`, `ellipse`, `line`, `polyline`, `polygon`,
   - paint servers: `linearGradient`, `radialGradient`, `stop`,
   - `clipPath` and `use` with fragment-only references (`href="#id"`, `xlink:href="#id"`, `url(#id)`),
   - presentation attributes from the allowlist in `rules/svg-logo-profile-v1.md`.
3. `v1` rejects `<style>` elements, `style` attributes, `mask`, `filter` and every filter primitive, in addition to everything PRD §16.5 already rejects. The diagnostic tells the author how to export with presentation attributes.
4. Defence in depth stays in place: bounded parsing with DTDs, entities and network disabled; fail-closed allowlist; the ADR-0009 limits; deterministic canonical output; and manifest hashes.
5. Masks, filters and CSS can be added later only through a new profile version (`flowframe-svg-logo/v2`) with its own ADR.
6. The generic icon pack (ADR-0013) uses the same `v1` profile.

## Consequences

If accepted:

- `tinycss2` and the CSS-allowlist validator for logos are removed from v0.1. The final-output CSS allowlist of ADR-0008 is unaffected.
- The P0.5 corpus shrinks to the accepted subset, the rejection cases and the limits.
- Logos that use CSS, masks or filters must be re-exported. This is a documented, one-time action for the project maintainer.
- The accessible name of the logo (ADR-0014 item 6) is attached to the image element, which avoids the cross-document references that inlining needed.
- Non-browser consumers in ADR-0022 may treat embedded SVG images differently. That is part of the verification below, not an assumption.
- Documents to change on acceptance: PRD §13.6 (logo embedding), §16.5 (profile contents); technical-spec §3 (stack), §12.3 steps 4 and 11, §13.2; implementation-plan P0.5, P4.4, P4.5.

## Verification

- P0.5: representative logos exported from Inkscape and Illustrator, once with presentation attributes, are accepted and render identically in every ADR-0022 consumer when embedded as an image.
- Forbidden constructs (`style`, `mask`, filters, scripts, external references, `foreignObject`, rasters) fail closed with actionable diagnostics.
- Repeat sanitization is byte-identical, and so are the manifest input and output hashes.
- An embedded logo with IDs equal to body IDs renders correctly without rewriting.
