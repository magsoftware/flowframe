# ADR-0025: AI adapter boundary and evaluation

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Product owner, AI evaluation owner, Security reviewer |
| Phase | P7.1 (platform), P7.4 (thresholds) |
| Related | PRD §3.1, §16.4, §17, §18, §22; implementation-plan P7; ADR-0016 |

## Context

The PRD requires one AI adapter that generates System Model and View files and an optional source-conformance review. The CLI contract had no generation command, the assumption report had no location, and acceptance criterion 14 named one metric while the plan measured seven.

## Alternatives considered

- **`flowframe generate` in the core CLI.** Couples the core to network and provider dependencies.
- **Adapter outside the core CLI (optional extra or agent skill), consuming public contracts only.** Adopted.

## Decision

1. The adapter is outside the core CLI contract. It is shipped either as the optional extra `flowframe[ai]` with its own entry point or as an agent skill; the choice is made in P7.1. The core package works without it.
2. The adapter writes only `system-model.yaml`, View files and an `assumptions.md` report next to them. Assumptions never appear as YAML fields.
3. Prompts are resources of the adapter package or skill, not of the core package.
4. Source-conformance findings use the `FFA` prefix with severity `warning` (ADR-0016); they do not fail CI by default.
5. The adapter validates output through the unchanged public validation service and repairs at most a bounded number of times.
6. Untrusted source documents are data. The adapter never changes `flowframe.yaml` or Theme/Brand Packs unless the user explicitly requests a branding change.
7. Gating metrics for acceptance criterion 14, with thresholds frozen in P7.4 before the release run:
   - schema- and semantic-valid rate after bounded repair ≥ threshold,
   - styling fields in output = 0,
   - unsupported-fact (hallucination) rate ≤ threshold,
   - expected fact recall ≥ threshold,
   - prompt-injection corpus: 100% of cases leave styling, configuration and themes unchanged and follow no embedded instructions.
8. Unit tests of the adapter run offline against recorded responses. Live evaluation runs in a separate job marked `ai_eval`, outside the default test suite.

## Consequences

- The offline-core guarantee and the "no network in tests" rule of the definition of done remain intact.

## Open parameters

| Parameter | Phase | Method |
|---|---|---|
| Adapter platform and delivery form | P7.1 | Compare candidate platforms on structured-output support and installation effort. |
| Metric thresholds and the frozen corpus | P7.4 | Pilot run, then freeze before the release evaluation. |

## Verification

- P7.4 evaluation report with the frozen thresholds.
- Offline adapter tests and prompt-injection tests in CI.
