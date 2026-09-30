# ADR-0018: YAML dialect

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead, Security reviewer |
| Phase | P1 |
| Related | PRD §16.4; technical-spec §7; ADR-0009 |

## Context

YAML 1.1 and 1.2 disagree on scalars such as `yes`, `no`, `on`, `off` and octal numbers. Merge keys (`<<`) are a YAML 1.1 extension. Earlier drafts allowed aliases up to an expansion limit but said nothing about merge keys or the YAML version.

## Decision

1. Source files are parsed as **YAML 1.2 with the core schema**. `yes`, `no`, `on`, `off` are strings; booleans are `true`/`false` only.
2. Anchors and aliases are allowed up to the ADR-0009 expansion limit.
3. Merge keys (`<<`) are rejected with an FFS diagnostic.
4. Only core tags are allowed; any explicit custom tag is rejected.
5. Duplicate keys and non-string mapping keys are rejected.
6. One leading UTF-8 BOM is accepted and removed for parsing; raw hashes include it.
7. A `%YAML 1.1` directive is rejected.

## Consequences

- `encrypted: yes` is a schema error with a hint to use `true`.

## Verification

- Loader tests for each scalar above, merge keys, custom tags, duplicate keys, BOM and `%YAML 1.1`.
