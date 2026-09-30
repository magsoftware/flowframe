# ADR-0011: Label composition and wrapping

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead, Accessibility and design reviewer |
| Phase | P0.3 (open parameters) |
| Related | PRD §9.3, §9.4, §10.8, §13.5, §13.10; technical-spec §10; ADR-0007 |

## Context

The PRD defines which information is visible per family and detail level, but not how it is composed into D2 labels. Order, separators and wrapping change every D2 golden, so they must be fixed before P3.

## Alternatives considered

- **Leave composition to each family generator.** Leads to inconsistent labels between families.
- **Rely on D2 automatic label sizing without wrapping.** Long labels produce very wide nodes and clipped layouts.
- **One framework-owned label grammar with deterministic wrapping.** Adopted.

## Decision

All label text is plain text, NFC-normalized and escaped by the D2 escaping module. Parts that are hidden by display flags or absent in the source are omitted together with their separator. The separator between inline parts is ` · ` (space, U+00B7, space).

**Element (architecture and flow nodes, sequence participants)**

```text
line 1: <label>[ [deprecated]]
line 2: [<technology>]            only when display.technologies is true and technology is set
```

**Boundary**: `<label> (<kind>)`; the system boundary uses `<system.label> (system)`.

**Relation (architecture and flow)**

```text
line 1: <semantic>[: <label>]     <label> only when display.relationLabels is true and label is set
line 2: <protocol> · <technology> · <payload> · transport encrypted|transport unencrypted
```

Line 2 parts follow `display.protocols`, `display.technologies`, `display.payloads` and `display.encryption`; the encryption text appears only when `encrypted` is set. An empty line 2 is omitted.

**Sequence message**: `[<n>. ]<label>[ (<protocol>)]`, where `<n>` is the one-based index among messages in scenario order when `display.stepNumbers` is true, and `<protocol>` is the explicit or inherited protocol when `display.protocols` is true. **Notes are not numbered.**

**Sequence note**: `<label>`.

**Wrapping**: each line is wrapped greedily at whitespace to the framework-owned `label-max-width` using the pinned font metrics of ADR-0007. An overlong run is split at grapheme boundaries. Text is never truncated. Wrapped lines are joined with D2 line breaks.

**Limits** (PRD §8.4) count only the source value, not the added `[deprecated]` marker or composed parts.

**Flow annotations**: stores and processing/messaging nodes are distinguished by their semantic shape and icon (ADR-0012), not by extra label text. Source/sink role annotations are not part of v0.1.

## Consequences

- One `generation/labels.py` module serves every family.
- Every relation label shows its semantic name, even with `display.relationLabels: false`.

## Open parameters

| Parameter | Phase | Method |
|---|---|---|
| `label-max-width` in CSS pixels | P0.3 | Render fixtures with long labels; choose the smallest width that keeps typical labels on at most two lines. |

## Verification

- D2 golden fixtures for every combination of display flags per family.
- Long and Unicode label fixtures in the P6.1 corpus.
