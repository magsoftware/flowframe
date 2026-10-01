# ADR-0030: One owning document per normative rule

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-10-01 |
| Owner | Product owner, Tech lead |
| Phase | P0.1 |
| Related | PRD §1, §25; technical-spec §1; implementation-plan §1; ADR-0001 |

## Context

ADR-0001 removed duplicated lists of open decisions. The normative content itself is still duplicated: the same rule is stated in full, or in a slightly different form, in two to four places. Examples in the current documents:

| Topic | Stated in |
|---|---|
| D2 executable resolution, pinning and checksum verification | README "Planned CLI", PRD §17, technical-spec §3, ADR-0006 |
| Output publication, control directory and recovery | README "Planned CLI", PRD §17.2, technical-spec §11.1, ADR-0010 |
| Manifest fields, resources and asset-processing records | PRD §16.1, technical-spec §15, ADR-0013, ADR-0020 |
| Decoration bands and width formula | PRD §13.6, technical-spec §12.3, ADR-0005 |
| Logo sanitizer processing | PRD §16.5, technical-spec §13.2, implementation-plan P0.5 |
| Exit codes, categories and precedence | PRD §17.1, technical-spec §16, ADR-0016 |

Each copy has to be kept consistent by hand. Several recent changes consisted mainly of re-aligning copies after review, and every further amendment repeats that cost. The PRD also carries implementation detail (processing order, internal file names, hashing layouts), so product readers have to read through design text to find behavior.

## Alternatives considered

- **Keep the current layout and rely on review to catch drift.** This is how the documents work today, and it is where the drift came from.
- **Give every normative rule one owning document; other documents link to it with at most one sentence of summary.** Proposed here.
- **Merge PRD and technical specification into one document.** Removes the duplication but mixes product decisions with implementation design, which ADR-0001 roles keep separate (Product owner and Tech lead).

## Proposed decision

1. **PRD** owns externally observable behavior: inputs and their schemas, vocabulary, selection semantics, CLI commands and options, exit codes, outputs and their contracts, accessibility and security guarantees as promises, and acceptance criteria. It does not describe processing order, internal modules, algorithms or internal resource file names.
2. **Technical specification** owns implementation design: module boundaries, processing pipelines, algorithms, internal resources, and how each PRD guarantee is achieved.
3. **ADRs** own the rationale and rejected alternatives of a decision, plus the values of their open parameters. When an ADR settles a rule, the rule is written once in the owning document (PRD or technical specification) and the ADR links to it. The ADR does not restate it.
4. **Implementation plan** owns sequencing, work packages and acceptance gates. It links to rules and does not restate them.
5. **README** is an overview for new readers. It contains no normative rules and links to the owning document.
6. A summary sentence that points to an owner is allowed. A second full statement of a rule, a table or an algorithm is not.

## Consequences

If accepted:

- One follow-up change, with no behavior change, moves every duplicated rule to its owner and replaces the copies with links. The table above is the minimum scope of that change.
- The PRD becomes noticeably shorter. Implementation detail moves to the technical specification, for example the sanitizer processing order, the publication state machine and the decoration formula.
- A future amendment touches one document per rule, plus links when a section moves.
- Applies equally to ADR-0026 to ADR-0029: if accepted, each one's rule is written in its owning document only.

## Verification

- After the follow-up change, a reviewer can take each topic in the table and find exactly one full statement of it.
- The documentation check of ADR-0001 is extended so that every cross-document link to a PRD or technical-specification section resolves.
