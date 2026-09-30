# ADR-0006: Pinned D2 distribution and verification

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead, Security reviewer |
| Phase | P0.1 (mechanism), P0.2 (open parameters) |
| Related | PRD §17; technical-spec §3, §11; ADR-0002, ADR-0003 |

## Context

Reproducible SVG requires one exact D2 build per platform. D2 is not bundled in the Python wheel. Integration tests need a verified D2 in CI from P3 onward.

## Alternatives considered

- **Bundle D2 inside platform wheels.** Simplifies installation but multiplies wheels and ties the FlowFrame release to D2 redistribution terms.
- **Accept any D2 with a matching version string.** Distribution rebuilds can differ in bytes and output.
- **Pin official release executables by per-platform SHA-256, install through an explicit helper.** Adopted.

## Decision

The verification and installation mechanism below is binding. The concrete D2 version, artifacts and hashes are open parameters filled in P0.2.

1. A package resource `resources/d2-pin.json` (schema `flowframe-d2-pin/v1`) records the D2 version and, per supported platform and architecture, the official archive URL, archive SHA-256, executable path inside the archive and executable SHA-256.
2. Executable resolution: `FLOWFRAME_D2` (absolute path; a relative path is a usage error, exit 2), otherwise the first `d2` on `PATH`.
3. Every rendering invocation verifies version and executable SHA-256 against the pin. Missing executable, version mismatch and checksum mismatch are distinct dependency diagnostics (exit 3) pointing to installation guidance.
4. The installer helper is `python -m flowframe.install_d2`, outside the core CLI contract. It supports `--from-archive PATH` (offline, verifies archive and executable hashes) and `--download` (explicit network access). It installs into `--dest DIR` and prints the `FLOWFRAME_D2` value. It is delivered in P1.8 and used by CI.
5. No FlowFrame command downloads D2 implicitly.
6. Changing the pin requires a FlowFrame release. Upgrade cadence: evaluated at every FlowFrame minor release; upgrades require reviewed golden and snapshot changes (ADR-0021).

## Consequences

- Homebrew or distribution-rebuilt D2 binaries are unsupported unless their bytes match the pin; documentation says so.
- Hashing the executable on every invocation costs a few milliseconds and is included in ADR-0023 measurements.

## Open parameters

| Parameter | Phase | Method |
|---|---|---|
| D2 version and supported platform/architecture list | P0.2 | Newest release that provides `d2 validate` with documented exits and renders all P0 fixtures offline on every supported platform. |
| Per-platform URLs and hashes | P0.2 | Download official artifacts, record hashes, verify licenses. |
| `d2 validate` exit behavior and failure modes | P0.2 | Exercise valid, syntax-invalid and crashing inputs. |

If P0.2 finds no D2 release providing `d2 validate` with usable exit behavior, a superseding ADR must change decision 3 or the validation boundary before P1.8 starts.

## Verification

- P0.2 acceptance in the implementation plan.
- P1.8: helper installs from a local archive without network; wrong archive hash fails.
- P3.4: missing executable, wrong version and wrong hash produce three distinct exit-3 diagnostics.
