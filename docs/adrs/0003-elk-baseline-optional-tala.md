# ADR-0003: ELK baseline layout engine, TALA optional

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead, Product owner |
| Phase | P0.1 (decision), P0.3 (open parameters) |
| Related | PRD §6.2, §12; technical-spec §11; ADR-0006 |

## Context

D2 supports several layout engines. ELK is bundled with D2 and has no separate commercial license. TALA is closed-source, separately installed and requires a commercial license for commercial use.

## Alternatives considered

- **Dagre (D2 default).** Bundled, but weaker for nested containers.
- **TALA as default.** Better for some non-hierarchical diagrams, but blocks offline open-source CI and adds licensing constraints.
- **ELK as default and only v0.1 engine; TALA post-MVP opt-in.** Adopted.

## Decision

1. v0.1 supports only `elk`. `render.layoutEngine` defaults to `elk`; `build --layout` and `render --layout` override it.
2. Precedence: `--layout` option > `render.layoutEngine` in the resolved configuration > `elk`.
3. An unsupported engine given on the command line is a usage error (exit 2). An unsupported engine in `flowframe.yaml` is a validation error (exit 1). A supported engine whose installation is missing is a dependency error (exit 3; relevant only post-MVP).
4. There is no fallback between engines, now or post-MVP, without a separate explicit contract.
5. TALA and `compare-layouts` remain post-MVP.

## Consequences

- Layout quality is limited to what ELK provides; the portable layout API stays small (`profile`, `direction`).
- The `hierarchical` and `compact` profiles need a pinned mapping to D2/ELK options.

## Open parameters

| Parameter | Phase | Method |
|---|---|---|
| D2/ELK options implementing `hierarchical` and `compact`, including body padding (`--pad`) | P0.3 | Render the P0 fixtures with candidate options; select the mapping that keeps nested containers readable; record the flags. |
| Direction mapping for `up/down/left/right` | P0.3 | Confirm D2 `direction` values with ELK on all fixtures. |

## Verification

- P0.3 fixtures render with the recorded flags.
- CLI contract tests: `--layout tala` → 2; `render.layoutEngine: tala` → 1.
