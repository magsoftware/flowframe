# Architecture Decision Records

This directory is the single register of FlowFrame architecture decisions. The PRD, the technical specification and the implementation plan reference decisions by ADR number instead of maintaining their own lists of open choices.

## States

| State | Meaning |
|---|---|
| Proposed | The question, alternatives and planned experiment are recorded; no decision is binding yet. Implementation that depends on it MUST NOT start. |
| Accepted | The decision is binding. It may list **open parameters** that a named phase fills in later without changing the decision itself. |
| Superseded | Replaced by a newer ADR, which is linked. The superseded file stays in place. |
| Rejected | Considered and not adopted. Kept for discoverability. |

Filling an open parameter is recorded as a dated amendment at the end of the same ADR. Changing a decision requires a new ADR that supersedes the old one, plus a PRD or technical-specification update in the same change.

## Roles

| Role | Responsibility |
|---|---|
| Product owner | Owns the PRD; accepts ADRs that change public product behavior or MVP scope. |
| Tech lead | Accepts technical ADRs; owns the technical specification and the implementation plan. |
| Security reviewer | Required reviewer for ADRs and changes touching input parsing, SVG/CSS sanitization, process execution, paths or publication. |
| Accessibility and design reviewer | Approves visual snapshot changes, contrast pairs, legend and accessibility templates. |
| AI evaluation owner | Owns the P7 evaluation corpus, thresholds and their freeze. |

One person may hold several roles. An ADR is accepted when the roles listed in its **Owner** field approve it.

## Template

```markdown
# ADR-NNNN: Title

| Field | Value |
|---|---|
| Status | Proposed / Accepted / Superseded by ADR-XXXX / Rejected |
| Date | YYYY-MM-DD |
| Owner | accepting role(s) |
| Phase | plan phase that must accept or complete it |
| Related | PRD §…, technical-spec §…, ADR-… |

## Context

Why a decision is needed and which constraints apply.

## Alternatives considered

Each alternative, what was tested and the result.

## Decision

The binding decision, stated normatively.

## Consequences

Positive and negative effects, follow-up work and affected documents.

## Open parameters

Only for Accepted ADRs whose values are measured later: parameter, owning phase, how it is determined.

## Verification

Tests, fixtures or experiments that prove the decision holds.
```

## Register

| ADR | Title | Status | Phase |
|---|---|---|---|
| [0001](0001-record-architecture-decisions.md) | Record architecture decisions | Accepted | P0.1 |
| [0002](0002-python-cli-and-d2-process-boundary.md) | Python CLI, packaging and D2 process boundary | Accepted | P0.1 |
| [0003](0003-elk-baseline-optional-tala.md) | ELK baseline layout engine, TALA optional | Accepted | P0.1 / P0.3 |
| [0004](0004-schema-authority-and-contract-versioning.md) | Schema authority and contract versioning | Accepted | P0.1 |
| [0005](0005-presentation-composition-outside-d2.md) | Global presentation composed outside D2 layout | Accepted | P0.1 |
| [0006](0006-pinned-d2-distribution.md) | Pinned D2 distribution and verification | Accepted | P0.1 / P0.2 |
| [0007](0007-svg-text-and-fonts.md) | SVG text representation, fonts and measurement | Accepted | P0.1 / P0.4 |
| [0008](0008-canonical-svg-output.md) | Canonical SVG output and normalization | Accepted | P0.3 |
| [0009](0009-global-limits.md) | Fixed implementation-wide limits | Accepted | P0.6 |
| [0010](0010-output-publication.md) | Output publication, control state and overwrite protection | Accepted | P0.7 |
| [0011](0011-label-composition.md) | Label composition and wrapping | Accepted | P0.3 |
| [0012](0012-shapes-and-token-application.md) | Element shapes and theme token application | Accepted | P0.3 / P0.6 |
| [0013](0013-generic-icon-pack.md) | Generic icon pack | Accepted | P0.3 / P0.6 |
| [0014](0014-svg-accessibility-metadata.md) | SVG accessibility metadata | Accepted | P0.3 |
| [0015](0015-legend-rendering.md) | Legend rendering | Accepted | P0.4 |
| [0016](0016-diagnostics-contract.md) | Diagnostics contract | Accepted | P1 |
| [0017](0017-command-stage-matrix.md) | Command stage matrix | Accepted | P1 |
| [0018](0018-yaml-dialect.md) | YAML dialect | Accepted | P1 |
| [0019](0019-default-value-resolution.md) | Single source of default values | Accepted | P1 / P0.6 |
| [0020](0020-theme-aggregate-hash.md) | Theme aggregate hash algorithm | Accepted | P1 |
| [0021](0021-golden-and-snapshot-policy.md) | Golden and snapshot policy | Accepted | P0.6 |
| [0022](0022-supported-svg-consumers.md) | Supported SVG consumers | Proposed | P0.1 |
| [0023](0023-performance-budgets.md) | Performance budgets | Proposed | P0.6 |
| [0024](0024-license-and-provenance-inventory.md) | License and provenance inventory | Accepted | P0.6 / P8 |
| [0025](0025-ai-adapter-boundary.md) | AI adapter boundary and evaluation | Accepted | P7.1 / P7.4 |
| [0026](0026-native-d2-decorations.md) | Decorations and legend placed by D2 | Proposed | P0.1 / P0.3 / P0.4 |
| [0027](0027-logo-image-embedding-and-minimal-profile.md) | Logo embedded as an image with a minimal v1 sanitizer profile | Proposed | P0.1 / P0.5 |
| [0028](0028-disposable-publication-state.md) | Output publication with disposable control state | Proposed | P0.1 / P0.7 |
| [0029](0029-v0-1-command-set.md) | Reduced v0.1 command set | Proposed | P0.1 |
| [0030](0030-single-owner-per-normative-rule.md) | One owning document per normative rule | Proposed | P0.1 |
| [0031](0031-flowframe-svg-writer-and-geometry-providers.md) | FlowFrame SVG writer with pluggable geometry providers | Proposed | P0.1 / P0.2–P0.4 |

## Mapping from former open-decision lists

| Former item | ADR |
|---|---|
| Pinned D2 version, upgrade cadence, installation and checksum strategy | 0006 |
| SVG text measurement, `<text>` versus paths, font formats, custom TTF faces | 0007 |
| Portable layout-profile mapping (`hierarchical`, `compact`) | 0003 |
| SVG normalization strategy | 0008 |
| Numeric limits for inputs, logos, XML, canvas, subprocess output and timeout | 0009 |
| Initial generic icon set, license and offline embedding | 0013 |
| Supported documentation renderers and browsers | 0022 |
| Committed versus CI-generated SVG | 0021 |
| Performance budgets | 0023 |
| Verification of PRD §10.8 display defaults | 0019 |
| Schema compatibility and deprecation policy | 0004 |
| First AI adapter platform and evaluation threshold | 0025 |
