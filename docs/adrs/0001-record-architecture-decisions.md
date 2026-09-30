# ADR-0001: Record architecture decisions

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead, Product owner |
| Phase | P0.1 |
| Related | PRD §21, §25; technical-spec §22; implementation-plan §5 |

## Context

The PRD, the technical specification and the implementation plan each maintained their own list of open Stage 0 decisions. The lists diverged in content and in timing ("Stage 0 or Stage 1" versus "Stage 0 only"). Several decisions also affect generated output and therefore golden tests, so they must be settled before dependent code is written.

## Alternatives considered

- **Keep decision lists inside each document.** Rejected: separate lists drift apart in content and timing.
- **Record decisions only in commit messages or issues.** Rejected: not discoverable offline and not reviewable next to the contracts.
- **One ADR register in the repository.** Adopted.

## Decision

1. All architecture decisions are recorded as ADRs in `docs/adrs/`, numbered `NNNN` in creation order, using the template and states in [README.md](README.md).
2. The PRD, the technical specification and the plan reference ADR numbers instead of listing open decisions.
3. Every ADR with phase P0.x MUST be Accepted before the P0 exit gate. An Accepted ADR may carry open parameters only when the owning phase and measurement method are named.
4. An ADR that changes public behavior or MVP scope requires Product owner approval and a PRD update in the same change.
5. Security-relevant ADRs (0006, 0008, 0009, 0010, 0013, 0018) additionally require Security reviewer approval.

## Consequences

- The P0 exit gate becomes checkable: every P0 ADR in the register is Accepted.
- Documents shrink because they no longer repeat decision lists.
- Superseded decisions remain in the directory.

## Verification

- A CI documentation check confirms that every ADR file listed in the register exists and that every ADR file is listed.
- Links from the PRD, the technical specification and the plan to `docs/adrs/` resolve.
