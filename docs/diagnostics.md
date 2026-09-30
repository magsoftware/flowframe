# FlowFrame diagnostic code registry

> **Status: stub.** The registry is populated in implementation-plan P2.4. Until then it contains no codes; the contract is defined in [ADR-0016](adrs/0016-diagnostics-contract.md).

This file is the authoritative list of FlowFrame diagnostic codes. Codes and their categories are public CLI behavior; message wording may change.

## Prefixes

| Prefix | Area |
|---|---|
| `FFC` | Project configuration and resolution |
| `FFS` | JSON Schema and source loading |
| `FFM` | System Model semantics |
| `FFV` | View and selection semantics |
| `FFT` | Theme, presentation and assets |
| `FFI` | Family IR invariants |
| `FFD` | D2 generation and validation |
| `FFR` | Renderer, SVG output and publication |
| `FFA` | AI adapter and source-conformance findings |
| `FFX` | Internal/unexpected failures |

## Entry format

Each code gets one row. Codes that exist only in prereleases are marked `prerelease-only` and removed before v0.1.

| Code | Severity | Category | Exit | Summary | Example | Status |
|---|---|---|---:|---|---|---|

## Codes

No codes are registered yet.
