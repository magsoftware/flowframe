# ADR-0029: Reduced v0.1 command set

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-10-01 |
| Owner | Product owner, Tech lead |
| Phase | P0.1 |
| Related | PRD §3.1, §17, §18.2, §22 (AC 18); technical-spec §10, §16, §17; ADR-0016, ADR-0017, ADR-0025 |

## Context

The v0.1 CLI contract (PRD §17) has six commands. Two of them add contract surface without adding capability:

- **`review` with its default modes `syntax`, `semantic` and `policy`** is defined to "produce the same findings/status as `validate --model ... --view ...`" (PRD §17, ADR-0017 rule 1). It is a second name for `validate`. It needs its own mode grammar, usage-error combinations, a differential test over the whole corpus (P2.9) and an AC 18 clause. The only capability `review` adds is source-conformance, which arrives with the AI adapter in P7.
- **`render`** renders a body-only SVG from a previously compiled D2 file. Because the D2 file is supplied from outside the build, it needs:
  - a theme fingerprint in the generated header,
  - a full policy lint of supplied input,
  - a separate classification of D2 syntax rejections as user validation errors (FFD, exit 1) rather than internal errors,
  - its own `--layout` and `--config` rules.

  The PRD itself states that `compile + render` is not equivalent to `build`. Its debugging value, seeing the D2 body without decorations, is already covered by `build --debug`, which keeps the raw renderer SVG (ADR-0008 item 3).

## Alternatives considered

- **Keep all six commands.** Matches the accepted contract and costs the work above.
- **v0.1 ships `validate`, `compile`, `build` and `version`; `review` exists only for source-conformance and arrives in P7; `render` is post-MVP.** Proposed here.

## Proposed decision

1. The v0.1 core commands are `validate`, `compile`, `build` and `version`.
2. Deterministic checking is `validate`. The deterministic `review` modes (`syntax`, `semantic`, `policy`) are not part of v0.1.
3. `review` is introduced in P7 for source-conformance only: `flowframe review --model MODEL --view VIEW [--config CONFIG] --source SOURCE...`. There is no `--mode` option. ADR-0025 is otherwise unchanged: FFA warnings, exit 3 when the adapter is missing, and a missing `--source` is a usage error.
4. `render` is post-MVP. If it returns, together with the hand-authored D2 compatibility mode mentioned in PRD §5, it gets its own ADR for the trust model of supplied D2.
5. The generated D2 header keeps the two lines that identify a FlowFrame artifact for overwrite protection (ADR-0010 item 5). The `flowframe-theme-sha256` line is dropped: `render` was its only consumer, and the manifest still records `theme.sha256`.
6. ADR-0017 keeps one validation service. Its differential test compares `validate --model M --view V` with `build` up to and including the family IR.

## Consequences

If accepted:

- Removed from v0.1:
  - P2.9 and its differential test,
  - the `--mode` grammar and its usage errors,
  - the render-specific header and fingerprint checks,
  - lint of externally supplied D2 and the FFD/exit-1 category for supplied artifacts,
  - `render --layout` precedence.
- `compile` remains, for D2 inspection and golden tests. It requires no D2 executable.
- Diagnostics and exit codes (ADR-0016) lose the cases that only `render` could produce. A D2 rejection inside `build` stays FFX (exit 6).
- Documents to change on acceptance: README "Planned CLI"; PRD §3.1, §3.2, §17, §17.1, §18.2, §22 AC 18; technical-spec §10, §16, §17; ADR-0016, ADR-0017 (columns `review` and `render`); implementation-plan P2.9, P3.7, P6.5, P7.5, traceability matrix.

## Verification

- CLI contract tests list exactly the v0.1 commands, and help output matches PRD §17.
- The differential `validate` and `build` test passes over the whole corpus.
- `build --debug` retains the raw body SVG in the diagnostic directory.
