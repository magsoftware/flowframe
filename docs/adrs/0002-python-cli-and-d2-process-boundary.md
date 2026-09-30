# ADR-0002: Python CLI, packaging and D2 process boundary

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-09-30 |
| Owner | Tech lead |
| Phase | P0.1 |
| Related | PRD §6.3; technical-spec §3, §11; ADR-0006 |

## Context

FlowFrame needs a portable CLI that runs locally and in CI, parses YAML, validates JSON Schema, emits D2 and post-processes SVG. D2 is written in Go and ships as a standalone executable.

## Alternatives considered

- **Go CLI linking the D2 library.** Tighter integration and a single binary, but couples FlowFrame to D2 internals, makes schema tooling and SVG/CSS sanitization harder, and raises the cost of contributions.
- **Node.js CLI.** Good SVG tooling, but weaker typed-schema and packaging story for the target users.
- **Python CLI invoking the pinned D2 executable through `subprocess`.** Adopted.

## Decision

1. Runtime: Python 3.12 or newer. CI runs Python 3.12 and 3.13.
2. Packaging: `pyproject.toml`, `uv` with a committed `uv.lock`, wheels built with Hatchling.
3. The package version is static in `pyproject.toml`; it is not derived from VCS metadata, so the version embedded in generated D2 headers does not change on every commit.
4. CLI framework: Typer, as a thin layer over the facade in `api.py`.
5. D2 is invoked as a separate process without a shell (ADR-0006). Direct Go integration is deferred.
6. Core dependencies: `ruamel.yaml`, `jsonschema` (Draft 2020-12), `lxml`, `tinycss2`. Text measurement and grapheme segmentation dependencies are selected by ADR-0007. Test dependencies include `pytest` and `hypothesis`. Static quality: Ruff and mypy (strict for core packages).
7. Supported release platforms: macOS and Linux. Advisory file locking uses POSIX `fcntl`; Windows is not a release gate.

## Consequences

- Business logic stays callable without a terminal and testable through the facade.
- Process startup cost of D2 is accepted for v0.1 and measured by ADR-0023.
- Rich terminal output from Typer MUST NOT leak into JSON-mode streams (ADR-0016).

## Verification

- P1.1: a clean environment installs the wheel; `flowframe version` works; CI passes on macOS and Linux with Python 3.12 and 3.13.
- P1 exit gate: installed wheel locates all packaged resources without repository-relative paths.
