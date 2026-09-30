# ADR-0023: Performance budgets

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-09-30 |
| Owner | Tech lead |
| Phase | P0.6 |
| Related | PRD §22; technical-spec §20 |

## Context

Performance budgets must come from measurements, not guesses, and become CI regression limits.

## Alternatives considered

- **No budgets.** Regressions go unnoticed.
- **Hard real-time targets.** Unrealistic across CI runners.
- **Generous regression limits derived from P0.2 baselines.** Recommended.

## Decision

To be accepted in P0.6: wall-clock and peak-memory limits for small and medium fixtures, measured separately for Python compilation and D2 rendering, on the reference CI runners. Limits are set at a documented multiple of the baseline.

## Consequences

- P6.6 enforces the limits in CI.

## Verification

- P6.6 benchmark job.
