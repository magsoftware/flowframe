# ADR-0014: SVG accessibility metadata

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead, Accessibility and design reviewer |
| Phase | P0.3 (hook proof) |
| Related | PRD §13.7, §22; technical-spec §12.3; ADR-0005, ADR-0007 |

## Context

Every final SVG needs a deterministic title, a top-level `<desc>` summary and descriptions for visible objects. The compositor must locate D2-rendered objects in SVG to attach per-object metadata. The earlier P0 plan checked hooks only for sequence output, and the summary format was undefined.

## Alternatives considered

- **Rely on D2 `tooltip`.** Output differs by consumer and may introduce interactive markup.
- **Only a top-level summary.** Does not satisfy per-object descriptions.
- **Compositor injection keyed by generated D2 identifiers.** Adopted, subject to the P0.3 hook proof.

## Decision

1. Applies to `build` output. `render` body SVG carries no summary or per-object metadata; documentation states this.
2. The root `<svg>` has `role="img"`, `aria-labelledby` pointing to a `<title>` and `aria-describedby` pointing to the top-level `<desc>`.
3. `<title>` is `presentation.title`, or the View ID when absent.
4. Top-level `<desc>` uses this English template, one sentence per line, in IR order:

   ```text
   <Family> view <view-id>.
   Purpose: <purpose>.                                  (only if set)
   Elements: <label> (<kind>)[ [deprecated]]; ...
   Relations: <source label> to <target label>, <semantic>, <interaction>; ...
   ```

   Sequence views replace the last two lines with:

   ```text
   Participants: <label> (<kind>); ...
   Steps: 1. <from label> to <to label>: <label>; note on <participant label>: <label>; ...
   ```

5. Each visible element, boundary and relation group receives `<title>` (its composed first label line) and `<desc>` (kind, technology, `[deprecated]`, and `description` when set).
6. The logo receives `<title>` from the theme asset `alt`.
7. All identifiers for metadata nodes come from the compositor's collision-safe ID allocator.

## Consequences

- The template is normative and documented in `rules/accessibility-rules.md`.
- If P0.3 cannot prove stable hooks, a superseding ADR must choose another mechanism before P4.9.

## Open parameters

| Parameter | Phase | Method |
|---|---|---|
| Hook mechanism mapping SVG groups to source IDs for nodes, containers, edges, participants, messages and notes | P0.3 | Inspect pinned D2 SVG for all three families; prove an injective, deterministic mapping. |

## Verification

- Golden tests for `<desc>` in each family.
- P6.4 checks in every ADR-0022 consumer that the accessibility tree exposes title and description.
