# FlowFrame — Executable Implementation Plan

## 1. Purpose

This plan decomposes [`prd.md`](prd.md) and [`technical-spec.md`](technical-spec.md) into ordered, independently verifiable work packages for FlowFrame v0.1.

The plan uses completion gates instead of calendar estimates. A phase is complete when its applicable automated gates pass in CI and required human reviews/ADRs are accepted. P0 is an experiment-and-decision gate, not a claim that ADR acceptance is executable in CI. Later work MAY begin in parallel where dependencies allow, but no milestone may be declared complete by deferring a failed gate.

The PRD is authoritative for product behavior. The technical specification is authoritative for the proposed implementation unless an accepted ADR changes it. All decisions are tracked in the ADR register [`docs/adrs/README.md`](adrs/README.md).

Every work package below lists its dependencies (**Dep**) and observable acceptance criteria (**Acc**). A package is done when its Acc items pass and the definition of done in section 4 holds.

---

## 2. Delivery principles

1. **Contracts before behavior.** Examples and tests are written against versioned schemas before compiler features depend on them.
2. **One vertical slice early.** Architecture/infrastructure proves the complete path before adding other diagram families.
3. **Deterministic core before AI.** AI adapters consume the same public contracts and cannot bypass validation.
4. **Presentation is global.** Theme, colors, logo and decoration placement are project-wide, never per-view overrides.
5. **Fail closed at boundaries.** YAML, filesystem paths, SVG/CSS and renderer processes are treated as untrusted.
6. **Every phase leaves a usable repository.** Main must pass formatting, types and the tests available at that phase.
7. **External upgrades are explicit.** D2, schemas, sanitizer profiles and golden snapshots change only through reviewed upgrades.
8. **One validation service.** `validate`, `review` and `build` share it and agree up to the family IR (ADR-0017).

---

## 3. Milestone map

| Plan phase | PRD stage | Result |
|---|---|---|
| P0 — Decisions and spikes | Stage 0 | External risks resolved and P0 ADRs accepted |
| P1 — Repository and contracts | Stage 1 | Installable CLI shell, validated public schemas, D2 installer |
| P2 — Loading, semantics and selection | Stage 1 | Safe, location-aware validation and deterministic review |
| P3 — Architecture vertical slice | Stage 1 | First complete deterministic build (alpha.1) |
| P4 — Design system and presentation | Stage 2 | Global branding, decorations, legend, accessibility and safe logos |
| P5 — Flow and sequence families | Stage 3 | All MVP families from reusable models |
| P6 — Hardening and release quality | Stages 2, 3 and 5 | Offline, secure and reproducible release candidate |
| P7 — AI adapter and evaluation | Stage 4 | One bounded, contract-driven AI workflow |
| P8 — Packaging and v0.1 release | Stage 5 | Installable, documented v0.1 |
| P9 — Optional capabilities | Stage 6 | Post-MVP work only |

### 3.1. Dependency chain

```text
P0 ─► P1 ─► P2 ─► P3 ─┬─► P4 ─────────────┐
                      └─► P5 (IR, projectors, generators)
                               └─ legend/accessibility parts wait for P4.8/P4.9
P4 + P5 ─► P6 ─┐
P5 contracts ─► P7 ─┴─► P8 ─► P9
```

- Flow and sequence IR design can start after P2.7 (selection), but their release gate depends on the generator and renderer from P3 and on P4.8/P4.9 for legend and accessibility.
- P4 theme schema work can start after P1; SVG composition integrates only after P3 produces body SVG.
- Deterministic `review` modes are delivered in P2.9; P7 adds only source-conformance.
- P7 adapter work needs the P5 contracts and may run alongside P6; P8 requires both P6 and P7.

---

## 4. Repository-wide definition of done

Every work item is done only when each criterion below that applies to it holds. A criterion applies once its subject exists: P0 spike code is exempt from this list; CI on macOS and Linux applies from P1.1; the coverage threshold applies to a listed package from the change that creates it; the diagnostic registry applies from P2.4; performance budgets apply from P6. A criterion that does not apply is stated as such in the issue, not silently skipped.

- production code is typed and passes Ruff and mypy (strict for `contracts`, `domain`, `validation`, `selection`, `projection`, `ir`, `generation`, `rendering`, `presentation`, `manifest`),
- line coverage of those packages stays at or above 90%,
- public behavior has unit or integration coverage,
- negative and boundary behavior is tested, not only the happy path,
- every new diagnostic has a stable code, a source-location assertion where applicable and an entry in `docs/diagnostics.md`,
- new public YAML/JSON structures have schema fixtures,
- output-affecting changes include reviewed D2 or SVG golden updates **and** a repeat-run determinism test,
- security-sensitive code has explicit rejection tests and Security reviewer approval,
- new dependencies, fonts, icons or assets are added to the relevant `LICENSES.md` and the provenance inventory,
- public contract changes have a changelog entry,
- documentation and examples match the implemented command names,
- tests in the default suite require no network access; tests marked `ai_eval` are excluded from it (ADR-0025),
- CI passes on macOS and Linux,
- at least one reviewer other than the author approves the change,
- performance budgets are not regressed,
- `git diff --check` and the project verification command pass.

Any intentional deviation from the PRD or technical specification requires an ADR and a documentation update in the same change. Review roles are defined in [ADR-0001](adrs/0001-record-architecture-decisions.md).

---

## 5. P0 — Decisions and technical spikes

### Objective

Remove uncertainty around D2, deterministic SVG generation, text measurement, logo safety, accessibility hooks and publication before building the production architecture.

### P0.1 — ADR framework and consumer list

Dep: —

Tasks:

- keep [`docs/adrs/README.md`](adrs/README.md) as the single decision register,
- review and accept ADR-0001 to ADR-0005,
- accept ADR-0022 (supported SVG consumers) before P0.3/P0.4 experiments.

Acc:

- every open decision has an ADR in the register with an owning phase,
- a documentation check verifies that all register links resolve,
- ADR-0022 lists consumers with versions and embedding modes.

### P0.2 — Pin and exercise D2 (ADR-0006)

Dep: P0.1

Tasks:

- select a D2 version available for supported macOS and Linux architectures,
- record official artifact URLs, licenses, archive and executable SHA-256 checksums,
- write a spike installation script (replaced by P1.8),
- render one architecture, one integration-flow and one sequence fixture with ELK,
- capture reported D2 and layout engine version behavior,
- test `d2 validate` and representative failure modes,
- test invocation with network unavailable and an allowlisted environment,
- measure startup, render time, memory and output size for small and medium fixtures (input to ADR-0023),
- document flags that affect generated SVG.

Acc:

- all three fixtures render offline,
- the chosen executable can be checksum-verified,
- the selected version provides `d2 validate` with documented exit behavior,
- the wrapper prototype distinguishes validation failure, renderer failure and timeout,
- version/flag data needed for the manifest is obtainable or its absence is explicitly modeled,
- ADR-0006 open parameters (version, platforms, artifacts, hashes, `d2 validate` behavior) are filled.

### P0.3 — Layout, SVG stability, shapes, labels and accessibility hooks

Dep: P0.2, ADR-0022

Tasks:

- render each fixture repeatedly in two isolated directories and compare D2 bytes, raw SVG and normalized SVG,
- inventory nondeterministic SVG fields, ordering and renderer CSS (ADR-0008),
- exercise long labels, Unicode, nested containers, all directionality values and mandatory self-messages/notes,
- render every element kind with its ADR-0012 shape inside nested containers and as a sequence participant,
- choose `label-max-width` from long-label fixtures (ADR-0011),
- prove embedded generic SVG icon data URIs render offline with one reusable declaration per unique asset, including multiple nodes/classes sharing it (ADR-0013),
- inventory D2 font data URIs and verify that plain labels produce no `foreignObject`,
- prove a deterministic mapping from SVG groups to source IDs for nodes, containers, edges, participants, messages and notes in all three families (ADR-0014),
- pin the body-padding flag and the `hierarchical`/`compact` profile mapping (ADR-0003),
- compare text layout and appearance in every ADR-0022 consumer,
- record baseline visual snapshots.

Acc:

- ADR-0008 open parameters filled: ignorable fields, serializer, CSS allowlist; canonicalization is idempotent on all fixtures,
- ADR-0003, ADR-0011, ADR-0012, ADR-0013 (embedding) and ADR-0014 open parameters filled,
- unsupported D2 constructs are listed; required self-messages cannot be downgraded to optional,
- no required MVP view needs manual coordinates.

### P0.4 — Presentation composition, text and legend

Dep: P0.3

Create one body SVG per family and exercise:

- title only; title plus subtitle,
- static footer; footer with `{source}`; no footer when `{source}` is required but source is absent,
- logo in each supported logo slot; title block and logo in distinct top slots,
- wider decoration than body; long title and Unicode source,
- slot conflict rejection, including a conflict created by the default title position,
- legend with every interaction and directionality sample.

Tasks:

- compare candidate deterministic text-measurement strategies and libraries,
- compare output-wide `<text>` with pinned/embedded fonts against text-to-path conversion,
- choose the built-in `flowframe-default` font family and document its script coverage,
- test the chosen strategy on both D2 body text and FlowFrame decoration text,
- test bundled fonts without host font discovery,
- derive top/bottom band and width expansion calculations,
- verify that legend samples drawn from shared line styles match rendered relations (ADR-0015),
- verify that translating the body preserves internal geometry,
- verify view-box and canvas behavior in ADR-0022 consumers.

Acc:

- ADR-0007 open parameters are filled: text policy, font family with script coverage, measurement library,
- ADR-0015 proof recorded,
- the band algorithm never clips or overlays the body in the fixture set,
- decoration placement is deterministic without browser automation,
- the same algorithm works on SVG emitted for every MVP family.

### P0.5 — Logo sanitizer spike

Dep: P0.1

Build a representative asset corpus containing:

- simple paths and groups; linear and radial gradients; clip paths and masks,
- internal `use` and `url(#id)` references,
- every proposed bounded filter primitive,
- a `style` block with allowed CSS,
- scripts and event attributes; external references and CSS imports,
- `foreignObject`, animation and raster images,
- excessive nesting, elements, path data and filter regions,
- duplicate or adversarial IDs and namespaces.

Tasks:

- write the exact `flowframe-svg-logo/v1` matrix in `rules/svg-logo-profile-v1.md`,
- validate candidate XML, CSS and path-data parsing approaches,
- prototype deterministic ID rewriting and reference normalization,
- select lossless number and namespace serialization,
- test Illustrator/Inkscape-style comments/metadata, allowlisted editor namespaces and fragment-only `xlink:href`,
- measure fixtures for ADR-0009 limits,
- confirm full-asset rejection with actionable diagnostics.

Acc:

- all allowed fixtures preserve expected appearance,
- all forbidden fixtures fail closed,
- repeat sanitization is byte-identical,
- original and sanitized hashes are stable,
- no regular-expression-only CSS parsing is used.

### P0.6 — Close release decisions

Dep: P0.2–P0.5

Decide and record:

- numeric limits (ADR-0009),
- icon set and license (ADR-0013),
- verification of PRD §10.8 display defaults on the corpus (ADR-0019); any change updates the PRD first,
- snapshot sets per platform and recommended consumer SVG policy, rasterizer for visual review (ADR-0021),
- performance budgets (ADR-0023),
- license inventory and SBOM approach (ADR-0024),
- shape confirmation (ADR-0012).

Acc:

- every listed ADR is Accepted or has its open parameters filled,
- `resources/limits/v1.json` values are recorded.

### P0.7 — Output publication spike (ADR-0010)

Dep: P0.2

Tasks:

- implement a throwaway state machine for staging, backup, journal and lock,
- inject failures at both rename crash windows and during cleanup,
- test lock contention between two processes,
- test forbidden output directories,
- run on APFS and ext4.

Acc:

- recovery restores or completes publication in every injected crash window,
- a competing writer fails immediately,
- journal format and fsync points recorded in ADR-0010.

### P0 exit gate

- every ADR with a P0 phase is Accepted, and open parameters owned by P0 are filled,
- PRD and technical specification reflect the decisions,
- spike code or fixtures are retained only when useful and clearly marked,
- the three representative diagrams render offline,
- presentation, sanitizer, accessibility hooks and publication designs are proven on real D2 SVG,
- no license, platform or dependency issue blocks P1.

---

## 6. P1 — Repository foundation and public contracts

### Objective

Create an installable, testable Python project with frozen v1 schemas, packaged built-in resources and a verified D2 installation path.

### P1.1 — Bootstrap the project

Dep: P0

Tasks:

- create `pyproject.toml` with Python 3.12 floor, static version and Hatchling build (ADR-0002),
- configure `uv.lock`, Ruff, mypy, pytest and `hypothesis`,
- add `src/flowframe` package and `flowframe` console entry point,
- add version reporting from installed package metadata,
- establish `tests/unit`, `tests/contract`, `tests/integration`, `tests/golden`, `tests/snapshots` and `tests/security`,
- add one verification command used locally and in CI,
- record the project CI provider; CI runs macOS and Linux with Python 3.12 and 3.13,
- ensure wheel and source distribution include all package resources.

Acc:

- a clean environment installs the local package,
- `flowframe version` succeeds,
- an installed wheel can locate all packaged resources without repository-relative paths.

### P1.2 — Schema conventions

Dep: P1.1

Tasks:

- use JSON Schema Draft 2020-12 with `$id` naming and versioning from ADR-0004,
- require `schemaVersion` discriminators,
- use closed objects unless a specific extension point exists,
- define shared primitives: ID grammar (model, View, theme, asset IDs), theme version, text rule and limits from PRD §8.4, URI-free local path,
- add a schema compilation test that resolves every reference offline,
- add a version registry in code,
- mark `default` annotations as informative and add the agreement test against `resources/defaults/v1.json` (ADR-0019).

Acc:

- every schema loads and resolves with the network disabled,
- unsupported versions produce one stable diagnostic,
- duplicate or conflicting `$id` values fail tests.

### P1.3 — Project Configuration schema

Dep: P1.2

Implement `flowframe-config.schema.json` for `flowframe-config/v1`: theme, `render.layoutEngine: elk`, `views` registry, logo asset and position, title position, footer enablement/template/position; no per-view color, font or asset overrides.

Acc — fixtures cover:

- minimal config and complete branding config,
- static footer and source footer,
- invalid positions and unknown properties,
- logo at `top-left` without an explicit title position (schema-valid; semantic conflict tested in P2.5).

### P1.4 — System Model schema

Dep: P1.2

Tasks:

- encode element, boundary, relation and scenario structures,
- encode empty-list defaults for omitted collections and the explicit membership example,
- encode the controlled vocabularies from the PRD,
- require explicit IDs plus `source` and `target` for every relation,
- require semantic/interaction; encode optional payload, metadata and message protocol/relationId,
- prohibit visual styling fields,
- encode the PRD §8.4 text limits and control-character rule.

Acc:

- fixtures include every kind and semantic value,
- every text limit has at-limit and over-limit fixtures,
- invalid styling and reference shapes are rejected.

### P1.5 — View Specification schema

Dep: P1.2

Tasks:

- encode family/subtype combinations, audience, detail, selection and portable layout,
- encode family-specific required/forbidden fields,
- restrict `select.kinds`/`exclude.kinds` to element kinds,
- forbid sequence select/exclude; allow scenarioId and optional participants,
- encode relation filters, depth 1–10,
- restrict `presentation` to `title`, `subtitle` and `source` with limits,
- reject theme, color, font, logo and position overrides.

Acc:

- fixtures for every forbidden field per family,
- a boundary kind in `select.kinds` fails.

### P1.6 — Theme, manifest, diagnostics and D2 pin schemas

Dep: P1.2

Theme schema: `flowframe-theme/v1` metadata; required color tokens from PRD §13.4 (without `primary`/`secondary`); optional numeric decoration tokens; mappings; exclusive `typography.fontSet`/`typography.fonts`; declared assets with media type, `alt`, path and bounds; required `licenses`.

Manifest schema: `flowframe-manifest/v1` per PRD §16.1, including the `builtInId` config variant, `assetProcessing` entries with `kind: logo`, `resources` with `kind: font|icon`, canonical SVG hash, optional `generatedAt`.

Diagnostics schema: `flowframe-diagnostics/v1` per ADR-0016.

D2 pin schema: `flowframe-d2-pin/v1` per ADR-0006.

Acc:

- all seven schemas pass positive and negative fixture suites,
- theme fixtures: reserved ID, `id` different from directory name, empty `typography`, both typography variants, missing one of four faces,
- manifest fixtures: path-based and `builtInId` configuration records.

### P1.7 — Built-in resources and examples

Dep: P1.6, ADR-0007, ADR-0013, ADR-0019, ADR-0009

Tasks:

- create immutable `flowframe-light` theme,
- add the `flowframe-default` font set bytes and `LICENSES.md`,
- add `mappings/v1.json`, `defaults/v1.json` and `limits/v1.json`,
- include an explicit package resource manifest,
- create minimum valid project, model and View fixtures,
- add one custom project theme fixture (rendered in P4).

Acc:

- installed wheel contains theme, fonts, mappings, defaults, limits and licenses,
- the defaults agreement test from P1.2 passes.

### P1.8 — D2 pin and installer helper (ADR-0006)

Dep: P1.6, ADR-0006

Tasks:

- add `resources/d2-pin.json` with per-platform data,
- implement `python -m flowframe.install_d2` with `--from-archive`, `--download` and `--dest`,
- use the helper in CI for every job that needs D2.

Acc:

- installation from a local archive works with network disabled,
- a wrong archive or executable hash fails with an actionable message,
- CI integration jobs use only helper-installed D2.

### P1.9 — Theme hash and command stage matrix

Dep: P1.6

Tasks:

- implement the ADR-0020 theme aggregate hash with a committed test vector,
- write the ADR-0017 command stage matrix into CLI help texts.

Acc:

- the test vector passes on macOS and Linux.

### P1 exit gate

- all seven schemas pass positive and negative fixture suites,
- package resources are present in an installed wheel,
- unknown properties and per-view style fields are rejected,
- examples parse as YAML and validate structurally,
- the helper installs D2 offline in CI,
- the CLI shell and verification command pass on macOS and Linux CI.

---

## 7. P2 — Safe loading, domain model, semantics and selection

### Objective

Turn structurally valid source files into immutable, location-aware domain objects, reject every invalid cross-document state and run selection so that `validate` and `review` report everything up to the family IR.

### P2.1 — Safe YAML loader (ADR-0018)

Dep: P1

Tasks:

- enforce UTF-8, BOM handling and the fixed byte limit,
- parse YAML 1.2 core schema only; reject `%YAML 1.1`, merge keys and custom tags,
- reject duplicate keys; bound nesting and aliases,
- normalize human text to NFC; keep raw bytes for hashing,
- convert values to JSON-compatible primitives,
- build the JSON Pointer to file/line/column map.

Acc:

- tests for `yes/no/on/off`, octal-looking values, merge keys, custom tags, duplicate keys, BOM, NFD input,
- raw hash of a BOM/NFD file differs from its normalized content while the built output is identical,
- alias-bomb and nesting fixtures fail with policy errors (5).

### P2.2 — Project and path resolver

Dep: P2.1

Tasks:

- explicit `--config` resolution; nearest-parent discovery bounded by the VCS root (`.git` file or directory); model directory only outside VCS,
- built-in config when absent,
- theme inventory, reserved built-in IDs (error whenever a reserved directory exists),
- traversal and symlink escape rejection; logical versus physical paths.

Acc:

- working directory does not change resolution,
- two nested configs resolve to the nearest one,
- a `.git` file (worktree/submodule) stops discovery,
- absent config cannot access project theme directories,
- out-of-root View → validation (1) with common-root guidance; symlink escape → policy (5),
- config and theme origin are observable for the manifest.

### P2.3 — Domain object construction

Dep: P2.1

Create frozen typed models for configuration and branding slots, theme tokens/mappings/assets, system elements/boundaries/relations/scenarios, View selection/presentation/family settings, and source references. Apply defaults only through `domain/defaults.py`.

Acc:

- domain objects contain no YAML-library nodes (type test),
- resolver unit tests for every family and detail combination.

### P2.4 — Diagnostics infrastructure (ADR-0016)

Dep: P1.6

Tasks:

- implement severity, code, category, message, locations, hint and debug,
- implement the ADR-0016 sort order, text renderer and JSON renderer,
- map JSON Schema paths back to YAML locations,
- implement redaction of absolute paths and exception details,
- populate the `docs/diagnostics.md` registry (the file exists as a stub) with `prerelease-only` marking,
- implement exit precedence 6 > 5 > 4 > 3 > 2 > 1.

Acc:

- JSON output validates against `flowframe-diagnostics/v1`; stdout is empty in JSON mode,
- a registry test fails for any code missing from `docs/diagnostics.md`,
- mixed-category precedence tests pass.

### P2.5 — Semantic validator

Dep: P2.3, P2.4

Implement and test:

- one model-wide ID namespace including all steps; unique View IDs across the registry and command line,
- element/boundary/relation/scenario reference resolution,
- containment cycles, valid parent types, `subsystem` ancestry, actor/external-system ancestry,
- relation combination matrix including the event+synchronous warning,
- scenario participants, notes restricted to elements, `relationId` direction and protocol inheritance/conflict, undirected relations cannot back messages,
- participant permutation exact and duplicate-free,
- theme token existence and mapping coverage, asset declaration and media types,
- branding slot conflicts after defaults, with a hint when a default caused the conflict,
- `subtitle` without `title`,
- View family/subtype, scenarioId, forbidden fields, traversal bounds, layout engine in configuration.

P2 owns structural/reference checks and path policy. Asset-content sanitization, contrast and full custom presentation validation integrate in P4.

Acc:

- every rule has at least one failing fixture with an asserted code and location,
- logo `top-left` with default title position fails with the default-conflict hint.

### P2.6 — Footer template parser

Dep: P2.4

Implement parser tokens (`Literal`, `Source`) per PRD §13.2, never string replacement.

Acc:

- `hypothesis` property tests over arbitrary brace sequences and Unicode,
- every PRD grammar rule has a fixture.

### P2.7 — Deterministic selection engine (moved from P3)

Dep: P2.5

Tasks:

- implement PRD §10.7 seeds, exclusions, relation defaults/filters, bidirectional traversal and ancestors,
- return inclusion reasons and stable ordering with ID tie-breakers,
- error on empty selections; warn for excluded explicit IDs, disconnected results and `systemBoundary` without members,
- implement integration-flow relation defaults and the undirected-dependency error.

Acc:

- A–B–C with excluded B proves traversal cannot reach C,
- empty filters, AND across criteria, relation-filtered traversal and bidirectional exploration are tested,
- property tests: `detail` and `audience` never change the selection,
- an unrelated undirected dependency does not block a flow view; a selected one fails with FFV.

### P2.8 — `validate` command

Dep: P2.5–P2.7

Tasks:

- implement model-only, model+view and exclusive theme-only forms following ADR-0017,
- run selection and (from P3.1 on) IR invariants when a View is given,
- use the PRD category-to-exit mapping,
- explicitly reject not-yet-supported P4 presentation features with `prerelease-only` diagnostics,
- ensure validation performs no rendering and writes no files.

Acc:

- CLI contract tests for success and every failure category,
- a filesystem-watch test proves no writes,
- repeated validation emits diagnostics in identical order.

### P2.9 — Deterministic `review` modes

Dep: P2.8

Tasks:

- implement `flowframe review` with modes `syntax`, `semantic` and `policy` over the same validation service,
- reject `--source` without `source-conformance` and `source-conformance` without `--source` (2); report the missing adapter (3).

Acc:

- differential test: default `review` equals `validate --model --view` on the whole corpus,
- review leaves model, View, configuration and artifacts unchanged.

### P2 exit gate

- every invalid reference and containment cycle in the corpus has an actionable location,
- footer grammar behavior exactly matches the PRD,
- path resolution is independent of working directory,
- `validate --view` reports empty selections and selection warnings,
- `flowframe validate` and deterministic `flowframe review` have CLI contract tests.

---

## 8. P3 — Architecture vertical slice

### Objective

Produce `diagram.d2`, canonical `diagram.svg` and `manifest.json` with staged, recoverable publication from one infrastructure view with the built-in theme and ELK.

### P3.1 — Architecture IR

Dep: P2.7

Define immutable IR types for node identity, label parts and semantic role, nested boundary groups, typed edges with directionality and label parts, portable layout hints and stable ordering.

Acc:

- IR validation tests cover duplicates, dangling references, invalid nesting and unsupported properties,
- `validate --view` now includes FFI invariants (ADR-0017).

### P3.2 — Infrastructure projector

Dep: P3.1

Map selected elements, boundaries and relations to the IR without duplicating elements; preserve source references; exclude all visual token values.

Acc:

- projection golden fixtures for each display flag combination.

### P3.3 — D2 writer and architecture generator

Dep: P3.2, ADR-0011, ADR-0012

Tasks:

- implement audited string/identifier escaping and injective ID mapping,
- implement `generation/labels.py` and `generation/line_styles.py`,
- apply ADR-0012 shapes and token application through built-in theme classes,
- emit the three-line generated header with theme fingerprint,
- fixed property ordering, nodes before relations, one trailing newline,
- materialize classes; emit no imports or host paths,
- implement the D2 lint rules of technical-spec §10,
- prohibit raw D2 from source documents.

Acc:

- byte-identical D2 golden tests,
- IDs equal to D2 keywords (`shape`, `style`, `near`, `label`, `classes`, `vars`) produce valid, distinct generated IDs,
- a fixture with warnings produces the same D2 as without them,
- label goldens cover `[deprecated]`, technology, protocol, payload and encryption text.

### P3.4 — D2 process wrapper

Dep: P1.8

Tasks:

- locate and verify pinned D2 (`FLOWFRAME_D2`, PATH),
- invoke without a shell in a new process group with an allowlisted environment,
- enforce the fixed timeout and output limits,
- validate before render; distinguish syntax rejection in build (FFX/6), in standalone render (FFD/1), and crash/timeout (execution/4),
- terminate the process group on SIGINT/timeout,
- reject partial output on non-zero exit.

Acc:

- missing executable, wrong version and wrong hash → three distinct exit-3 diagnostics,
- relative `FLOWFRAME_D2` → 2,
- no child process survives SIGINT or timeout,
- partial output is never published.

### P3.5 — Body SVG verification and canonicalization

Dep: P3.4, ADR-0008

Tasks:

- parse with hardened XML settings,
- reject external references, unsafe nodes and CSS outside the ADR-0008 allowlist,
- verify a finite view box and canvas limits,
- canonicalize and add the FlowFrame marker comment,
- expose body bounds for composition.

Acc:

- pinned-D2 output passes; fixtures with `<script>`, external `url()` or disallowed CSS fail (5),
- canonicalization is idempotent.

### P3.6 — Manifest collector and writer

Dep: P3.3–P3.5, P1.9

Tasks:

- record source/config/theme hashes (ADR-0020) and schema versions,
- record D2/layout identity and flags, font and icon resources,
- record D2 and canonical SVG hashes,
- serialize stable JSON validated against the manifest schema,
- omit `generatedAt` unless a valid `SOURCE_DATE_EPOCH` is supplied,
- omit `assetProcessing` when no logo was sanitized.

Acc:

- invalid `SOURCE_DATE_EPOCH` (negative, non-integer, out of RFC 3339 range) → 2,
- no absolute paths in the manifest,
- the D2 header fingerprint equals `theme.sha256`.

### P3.7 — `compile` and low-level `render`

Dep: P3.3–P3.5

Tasks:

- compile without D2; render only generated, linted D2 with a matching theme fingerprint,
- apply `render --layout` precedence and explicit `--config`,
- implement single-file overwrite protection (ADR-0010),
- implement PRD CLI flags, text/JSON diagnostics and debug behavior.

Acc:

- `compile` succeeds with an empty PATH,
- missing header or fingerprint mismatch → 1,
- `--output` pointing to an input file, or to an existing non-artifact file, → 5 with the file unchanged.

### P3.8 — Recoverable `build`

Dep: P3.6, P3.7, ADR-0010

Tasks:

- implement the technical-spec §11.1 state machine,
- reject forbidden output directories,
- accept absent/empty targets and intact previous FlowFrame outputs; reject foreign files, modified artifacts and symlinks,
- preserve the old target until verification succeeds; attempt rollback and report unsuccessful recovery,
- retain at most one previous output and one staging set.

Acc:

- tests for concurrent writers (4), both rename crash windows, first build into an empty directory, rollback failure, cleanup failure, interruption (130),
- every forbidden output directory → 5,
- a build writes nothing outside the target and its control directory.

### P3.9 — Alpha.1 example

Dep: P3.8

Add the payments example with minimal explicit config (`theme: flowframe-light`, footer disabled), `display.legend: false`, no logo, custom theme or presentation metadata:

```text
examples/payments/flowframe.yaml
examples/payments/system-model.yaml
examples/payments/infrastructure-view.yaml
```

One documented command creates:

```text
build/payments/infrastructure-overview/
├── diagram.d2
├── diagram.svg
└── manifest.json
```

Acc:

- the command works in a clean offline environment,
- two runs produce byte-identical D2 and SVG,
- unsupported presentation requests fail with `prerelease-only` diagnostics.

### P3 exit gate — v0.1-alpha.1

- a clean offline environment builds the payments infrastructure view,
- repeat runs produce byte-identical D2 and canonical body SVG,
- manifest validates and contains the pinned toolchain identity,
- renderer failures before publication preserve the previous complete output; publication failures expose documented rollback/recovery behavior,
- no source document can inject D2 or visual style,
- CI passes on macOS and Linux.

---

## 9. P4 — Design system, branding and presentation

### Objective

Apply one project-wide Theme/Brand Pack and safely add title, subtitle, source footer, logo, legend and accessibility metadata without modifying semantic layout.

Packages are ordered by dependency.

### P4.1 — Custom theme processing

Dep: P3

Tasks:

- load exactly one complete theme pack; no partial merging with built-in packs,
- reuse P2 resolution and validation; resolve missing mappings from `mappings/v1.json`,
- resolve only declared local assets,
- reuse the ADR-0020 hash for custom themes,
- record `project` or `built-in` origin.

Acc:

- custom theme fixture builds; its hash matches the manifest and D2 header,
- a declared but unused asset changes the hash.

### P4.2 — Contrast validator (ADR-0012)

Dep: P4.1

Implement `presentation/contrast.py` for the PRD §13.9 pairs from resolved mappings, with logo exemption.

Acc:

- built-in theme passes every pair,
- a custom theme failing one pair produces an FFT error naming both tokens,
- contrast checks run in `validate`, `compile` and `build` (ADR-0017).

### P4.3 — Rendering tokens and mappings

Dep: P4.1, P4.2

Render every token according to PRD §13.9 for built-in and custom themes; reject unknown token references.

Acc:

- D2 goldens show each mapped token in its specified role,
- a mapping override changes only the mapped color.

### P4.4 — Normative sanitizer profile

Dep: P0.5

Finalize `rules/svg-logo-profile-v1.md`: allowed elements, attributes, CSS, gradient/clip/mask/reference rules, filter matrix, ADR-0009 limits, rejection matrix, canonicalization and ID rewriting, conformance examples.

Acc:

- the document and implementation share profile fixtures; a test fails if a fixture is referenced by only one of them.

### P4.5 — Production logo sanitizer

Dep: P4.4

Tasks:

- byte cap before parsing; XML with DTD/entities/network disabled,
- element, nesting, path and filter-region limits,
- CSS parsing with `tinycss2`,
- reference graph validation; reject external/data URLs and raster images,
- fail on unknown content; remove only allowlisted inert metadata/comments,
- deterministic ID rewriting, canonical bytes and both hashes,
- expose finite intrinsic bounds to the compositor.

Acc:

- the complete P0 corpus passes expected allow/reject results,
- sanitizer results are byte-identical across repeated platform runs,
- manifest records profile, implementation/version and both hashes with `kind: logo`.

### P4.6 — Generic icons (ADR-0013)

Dep: P4.5, ADR-0013

Tasks:

- package the icon set with `index.json` and `LICENSES.md`,
- sanitize icons at package build and commit canonical bytes,
- verify icon hashes at runtime; embed each icon once in D2,
- extend the D2 lint to accept only indexed icon data URIs.

Acc:

- package-build test: every icon passes the sanitizer and matches its hash,
- tampered bytes, missing entry and missing file fail,
- updated P3 D2 goldens are reviewed and accepted.

### P4.7 — Text metrics and fragments (ADR-0007)

Dep: ADR-0007

Tasks:

- implement `presentation/text_metrics.py` shared by label wrapping and decorations,
- render title/subtitle as one block; static and source footer tokens as escaped plain text,
- omit a source-dependent footer when source is absent; keep a static footer,
- wrap at whitespace with grapheme fallback under the maximum width.

Acc:

- wrapping goldens for Latin, accented, CJK and emoji text; uncovered scripts behave as documented,
- label wrapping in D2 and decoration wrapping use the same metrics (unit test).

### P4.8 — Legend (ADR-0015)

Dep: P4.3, P4.7

Implement `presentation/legend.py` using `generation/line_styles.py`.

Acc:

- legend entries equal the styles present in the IR, in vocabulary order,
- an empty legend occupies no band,
- snapshot comparison of samples against rendered relations.

### P4.9 — SVG accessibility metadata (ADR-0014)

Dep: P4.7, P0.3 hook proof

Tasks:

- implement `presentation/accessibility.py`: root ARIA attributes, `<title>`, ADR-0014 `<desc>` template, per-object metadata,
- document the template in `rules/accessibility-rules.md`,
- verify `[deprecated]` in labels and descriptions.

Acc:

- `<desc>` goldens for architecture; P5 extends them to flow and sequence,
- missing title uses the View ID,
- metadata IDs never collide with body or logo IDs.

### P4.10 — Decoration compositor

Dep: P4.5, P4.7–P4.9

Implement in this order: read verified body bounds; measure title block, logo, legend and footer; fit logo ("contain"); compute bands; compute slot widths per PRD §13.6; translate body once; place fragments with collision-safe IDs; inject accessibility metadata; canonicalize and verify.

Acc — tests cover:

- no decorations; every individual decoration; all valid slot combinations,
- invalid same-slot title/logo combination, including via defaults,
- absent optional metadata; long and Unicode text,
- logo smaller and larger than its box; logo IDs colliding with body IDs,
- narrow and wide bodies; body and decorations at canvas limits,
- automated bounds: decoration and legend bounds never intersect body bounds.

### P4.11 — Global consistency and manifest provenance

Dep: P4.10

Tasks:

- reject per-view theme, raw style, font, logo and position fields,
- build multiple infrastructure fixtures with one project theme and prove identical theme hashes,
- add a custom branded fixture with logo, title and source footer,
- extend the P3 collector with configuration identity, theme identity, sanitizer identity, asset hashes and final SVG hash.

Acc:

- a configuration change affects every rebuilt diagram,
- the manifest contains sufficient information to reproduce asset processing.

### P4.12 — Project-wide `build --all`

Dep: P4.11, P2.8

Tasks:

- implement `build --all` over the `views` registry with one model/config/theme,
- validate all views up to the IR before rendering any,
- publish to `<output-dir>/<view-id>/` in sorted order with P3 per-view recoverable replacement,
- report partial progress on render/publication failure.

Acc:

- a semantic error in the third view publishes nothing,
- a render failure in the third view leaves two published views and reports them,
- duplicate View IDs and empty registry fail.

### P4.13 — Rules documents

Dep: P4.2–P4.9

Create `rules/visual-guidelines.md`, `layout-rules.md` (including the 7–9 element guideline, documentation only), `naming-rules.md`, `diagram-types.md` and `accessibility-rules.md`.

Acc:

- each normative statement links to its test or fixture.

### P4 exit gate

- built-in and custom theme fixtures render deterministically,
- logo/title/footer/legend never cover or clip the body (automated bounds),
- static and `{source}` footer semantics match the PRD,
- every unsafe/over-complex logo fixture is rejected,
- built-in and custom themes pass all contrast pairs,
- manifest contains sufficient information to reproduce asset processing.

---

## 10. P5 — Integration-flow and sequence families

### Objective

Complete the three-family MVP while reusing the same System Model, validation, theme and output pipeline.

### P5.1 — Flow IR and projector

Dep: P2.7, P3.3

Tasks:

- define flow nodes and directed/bidirectional edges with label parts,
- map relations, protocols and payloads; distinguish stores by shape and icon only,
- reuse the single selection algorithm with integration-flow defaults.

Acc:

- small, medium and nested-boundary fixtures,
- deterministic projection and D2 goldens,
- differential test: `validate --view` and `build` report identical selection diagnostics for flow views.

### P5.2 — Flow generator

Dep: P5.1; P4.8 and P4.9 for legend and accessibility

Tasks:

- emit flow-specific D2 through the shared writer and label module,
- render `[deprecated]` markers and accessible descriptions,
- render and snapshot with ELK.

Acc:

- label goldens with `display.relationLabels: false` still show semantic names,
- flow `<desc>` goldens,
- snapshots approved.

### P5.3 — Sequence IR and projector

Dep: P2.7

Tasks:

- derive ordered participants (first occurrence or explicit permutation),
- implement messages and notes in source order,
- validate relation-backed messages; keep boundary IDs out of participant references.

Acc:

- ordering fixtures: repeated participants, note-only participants, explicit permutation,
- ordering independent of model map order (property test).

### P5.4 — Sequence generator

Dep: P5.3; P4.8 and P4.9 for legend and accessibility

Tasks:

- emit D2 sequence constructs,
- solid single-arrow messages, reverse messages for responses, self-messages,
- message numbering per PRD §13.10 (notes not numbered),
- `relationId` never imports relation appearance,
- optional sequence legend with only actual constructs,
- document D2 limitations accepted in ADRs.

Acc:

- goldens for numbering on and off, self-messages, long labels,
- sequence `<desc>` goldens with ordered steps.

### P5.5 — Cross-family reuse example

Dep: P5.2, P5.4, P4.11

Extend the payments model with one integration-flow view, one sequence scenario and view, no duplicated inventory, shared theme and branding.

Acc:

- `build --all` publishes all three views from one model.

### P5 exit gate

- at least five fixtures per MVP family pass schema, semantic, D2 golden and SVG snapshot checks,
- one System Model builds architecture, flow and sequence outputs,
- all outputs use the same resolved theme and branding policy,
- sequence order and flow direction remain stable on repeated builds,
- family-specific invalid inputs produce source-located diagnostics.

---

## 11. P6 — Hardening and release quality

### Objective

Turn feature-complete behavior into a secure, portable, diagnosable release candidate.

### P6.1 — Golden corpus completion

Dep: P5

Corpus: five views per family; small, medium and limit-size cases; nested boundaries; long and Unicode labels; explicit null icons and invalid packs; omitted presentation metadata; custom theme with every decoration; invalid contracts and semantic states.

Acc:

- corpus inventory document lists every fixture and its purpose.

### P6.2 — Reproducibility

Dep: P6.1, ADR-0021

Tasks:

- run identical builds in separate absolute paths, twice per supported platform,
- compare D2 bytes, canonical SVG bytes within the toolchain equivalence class and manifests,
- verify no absolute paths, random IDs or locale-dependent values leak,
- set locale/timezone explicitly in CI.

Acc:

- SVG snapshot sets follow the ADR-0021 decision (shared or per platform),
- reproducibility report committed.

### P6.3 — Security suite

Dep: P6.1

Tasks:

- YAML alias/nesting/input-size attacks at and above every ADR-0009 limit,
- path traversal, symlink escape and forbidden output directories,
- single-file overwrite attempts,
- malicious labels aimed at D2 injection,
- XML entities and DTDs; SVG scripts, events, external URLs, CSS imports,
- excessive SVG/filter/path complexity,
- renderer timeout, output flooding and orphan processes,
- environment influence (non-allowlisted variables),
- stale/partial artifact publication.

Acc:

- every case fails closed with the documented category.

### P6.4 — Accessibility and visual review

Dep: P6.1, ADR-0022

Tasks:

- run the contrast validator on the complete corpus,
- run the automated AC9 bounds checks (decorations versus body, labels inside shapes, sibling nodes disjoint),
- run the AC10 checks (distinct shapes/border styles per kind, distinct stroke patterns per interaction, semantic names present),
- generate grayscale snapshots with the ADR-0021 rasterizer,
- verify the text representation and accessibility tree in every ADR-0022 consumer,
- conduct and record a human review of the full corpus.

Acc:

- automated checks pass; the recorded review has no unresolved blocker; accepted renderer quirks are listed.

### P6.5 — CLI contract completion

Dep: P5

Tasks:

- test every command help page,
- test exit codes 0–6 and 130, including a synthetic exception and mixed-category precedence,
- validate every JSON-mode stderr against `flowframe-diagnostics/v1` and assert empty stdout, including with `--debug` and `PYTHONWARNINGS=always`,
- hide stack traces outside debug mode,
- SIGINT in every publication phase: rollback/cleanup and reporting of any retained backup.

Acc:

- the contract suite covers every row of PRD §17.1.

### P6.6 — Performance gates (ADR-0023)

Dep: P6.1

Benchmark small and medium corpus builds, separating Python compilation from D2 render time; enforce ADR-0023 limits in CI; document splitting large diagrams into views.

Acc:

- CI benchmark job enforces the budgets.

### P6 exit gate — release candidate

- all PRD acceptance criteria unrelated to P7 AI and P8 packaging pass,
- the full default suite runs offline,
- supported-platform reproducibility is documented and tested,
- security corpus fails closed,
- recorded visual review has no unresolved blocker,
- performance stays within accepted budgets.

---

## 12. P7 — AI adapter and evaluation (ADR-0025)

### Objective

Add one optional adapter that produces public structured contracts without receiving control of styling or deterministic output.

### P7.1 — Adapter boundary

Dep: P5, ADR-0025

Tasks:

- select the platform and delivery form (optional extra or agent skill) and record it in ADR-0025,
- define provider-neutral request/result types,
- separate source-document extraction from model/view generation,
- accept structured output only; keep credentials and network logic outside the core package.

Acc:

- the core package installs and passes its suite without AI dependencies.

### P7.2 — Generation prompts and rules

Dep: P7.1

Create System Model and View generation prompts, `assumptions.md` report format, rules forbidding raw styles and invented claims, ID-reuse instructions, source-conformance prompt, `review-diagram.md`, `compare-with-source.md`, `rules/ai-generation-rules.md` and one generation skill or equivalent adapter. Prompts are adapter resources. `simplify-view.md`, refactor skills and visual review are P9.

Acc:

- prompts reference only public schema values; a lint test rejects styling terms in prompt examples.

### P7.3 — Validation-repair loop

Dep: P7.2, P2.4

Tasks:

- validate generated YAML through the normal pipeline,
- return `flowframe-diagnostics/v1` output to the adapter,
- bound repair attempts,
- never edit project configuration or Theme/Brand Packs unless explicitly authorized,
- show the final source diff for human review.

Acc:

- offline tests with recorded responses cover success, repair and repair exhaustion.

### P7.4 — Evaluation corpus

Dep: P7.3

Measure the ADR-0025 gating metrics (valid rate after repair, zero styling fields, hallucination rate, fact recall, prompt-injection corpus) plus ID stability and view family/selection correctness. Run a pilot, then freeze the corpus and thresholds with AI evaluation owner approval before the release run. Do not adjust thresholds after observing release results.

Acc:

- the frozen corpus includes prompt-injection sources,
- the evaluation report shows every gating metric at or beyond its threshold.

### P7.5 — Source-conformance review

Dep: P7.3, P2.9

Tasks:

- implement `review --mode source-conformance` with explicit `--source` inputs,
- report findings as `FFA` warnings,
- report a missing adapter as exit 3.

Acc:

- review stays read-only in every mode,
- findings validate against the diagnostics schema.

### P7 exit gate

- one adapter generates models and views that enter the unchanged compiler,
- invalid AI output cannot bypass validation,
- every gating metric meets its frozen threshold,
- normal generation never emits theme/style values; explicit branding edits are a separate authorized workflow,
- assumptions and source gaps remain visible to the user.

---

## 13. P8 — Packaging, CI and v0.1 release

### Objective

Ship a verifiable package and a documented offline workflow.

### P8.1 — Distribution

Dep: P6, P7

Tasks:

- build wheel and source distribution with schemas, built-in theme, fonts, icons, pin and licenses,
- test the P1.8 installer helper from the release artifact, online and offline,
- verify package resource access outside the source checkout,
- generate a CycloneDX JSON SBOM (ADR-0024).

Acc:

- installation from the release artifact passes on macOS and Linux,
- SBOM generated and attached to the release.

### P8.2 — CI matrix

Dep: P8.1

Required jobs: formatting/linting; static typing; unit and contract tests; integration tests with pinned D2; security tests; deterministic/golden tests; package build and installed-wheel smoke test; SBOM generation; performance benchmarks; Python 3.12 and 3.13.

At least macOS and Linux run the installed-wheel smoke test. Expensive visual suites MAY run once per pinned renderer platform if ADR-0021 established cross-platform equivalence.

Acc:

- all jobs required for merging to main.

### P8.3 — Consumer documentation

Dep: P8.1

Document:

- installation of FlowFrame and D2 with the helper or offline archive; distribution rebuilds are unsupported unless bytes match the pin,
- project structure, including a common-root `flowframe.yaml` for sibling `model/` and `views/`,
- creating `flowframe.yaml`, a global theme, model and View files,
- building one or all diagrams; forbidden output directories,
- title, subtitle, static/source footer, logo and legend behavior,
- asset and font restrictions, including documented script coverage of the default font,
- output/manifest interpretation, header fingerprint equal to `theme.sha256`, failure categories,
- ordinary output directories, ignoring private `.<name>.flowframe/` state, `.gitattributes` and formatter exclusions (ADR-0021), bounded retention and recovery,
- CI examples for GitHub Actions and GitLab CI,
- upgrades and golden snapshot review,
- troubleshooting by diagnostic code.

Acc:

- every documented command is executed by a documentation test.

### P8.4 — Release checklist

- all PRD MVP acceptance criteria pass,
- schemas and CLI help are versioned,
- package contains required assets and licenses; manual license review signed off (ADR-0024),
- release artifacts have checksums and an SBOM,
- changelog and known limitations are published, including font script coverage,
- installation is tested from the release artifact,
- no open critical security or data-loss issue remains,
- no `prerelease-only` diagnostic codes remain,
- exact D2/sanitizer/theme versions are recorded.

### P8 exit gate — v0.1

A new consumer repository can install FlowFrame and the pinned D2, create a project-wide branded configuration and produce architecture, flow and sequence SVGs offline from validated YAML using documented commands.

---

## 14. P9 — Post-MVP backlog

These items do not block v0.1:

- optional TALA adapter and `compare-layouts` with two or more explicitly selected engines and no fallback,
- dark built-in theme,
- additional architecture and flow subtypes (`flowframe-view/v2`),
- vendor icon packs with independent licenses,
- PNG, PDF and PPTX outputs,
- Windows release support,
- schema migration tooling,
- `validate --all` and `review --all`,
- element-count guideline warnings,
- `select.within` boundary-scoped selection,
- flow source/sink annotations,
- glyph-coverage validation,
- machine-readable SPDX license inventories,
- Python facade entry points for `render` and `review`,
- advanced visual-analysis loop, `review --mode visual`, simplify/refactor prompts and skills with a defined quality metric,
- public extension/plugin API,
- performance cache with complete content-addressed keys.

Every post-MVP item requires its own contract and security review. TALA MUST remain opt-in and MUST never become an implicit fallback.

---

## 15. Recommended issue breakdown

Create one tracking epic per phase and one issue per numbered work package. Each issue SHOULD contain:

```text
Context:
  Links to exact PRD, technical-spec and ADR sections.

Scope:
  Concrete implementation and files affected.

Out of scope:
  Explicitly deferred behavior.

Acceptance:
  The package's Acc items: observable checks, fixtures and commands.

Dependencies:
  The package's Dep items: blocking issue IDs and accepted ADRs.

Security/reproducibility:
  Boundary conditions introduced by the change.

Artifacts:
  Schema, code, fixtures, diagnostics registry entries and documentation expected.
```

Do not group unrelated schema, renderer and presentation changes into one review. Small contract-first changes keep generated diffs and security decisions reviewable.

---

## 16. Traceability matrix

| PRD requirement (section) | Work packages | Main verification |
|---|---|---|
| AC1 versioned contracts (§6.3, §16.1) | P1.2–P1.6 | Schema fixture suites |
| AC2 actionable diagnostics (§15.3) | P2.4, P2.5 | Invalid-model corpus with location assertions |
| AC3 deterministic D2 and SVG (§16.1) | P3.3, P3.5, P6.2 | Byte goldens, repeat builds in two paths |
| AC4 offline ELK rendering (§16.2) | P3.4, P6 | Network-disabled CI job |
| AC5, AC6 global theme (§13) | P4.1, P4.11, P5.5 | Cross-view/family consistency tests |
| AC7 no raw styles in D2 (§13.1) | P3.3, P4.11 | Lint and policy fixtures |
| AC8 same-model families (§10) | P5 | Cross-family golden corpus |
| AC9 no clipping or overlap (§13.6) | P4.10, P6.4 | Automated bounds checks, recorded review |
| AC10 grayscale (§13.5, §13.9, §13.10) | P3.3, P6.4 | Shape/pattern distinctness checks, grayscale review |
| AC11 contrast (§13.9) | P4.2, P6.4 | Contrast validator |
| AC12 provenance (§16.1) | P3.6, P4.11 | Schema and hash assertions |
| AC13 renderer failures (§17.1) | P3.4, P6.5 | Exit-code contract tests |
| AC14 AI quality (§18) | P7.4 | Frozen evaluation corpus |
| AC15 `build --all` (§17) | P4.12, P5.5 | Installed CLI project build |
| AC16 security (§16.4, §16.5) | P6.3 | Rejection corpus |
| AC17 accessibility metadata (§13.7) | P4.9, P5.2, P5.4 | `<desc>` goldens, consumer checks |
| AC18 review and exit codes (§17) | P2.9, P7.5, P6.5 | Differential and contract tests, JSON schema validation |
| AC19 publication safety (§17.2) | P0.7, P3.7, P3.8 | Crash-window, rollback and overwrite tests |
| AC20 D2 pin and installer (§17) | P1.8, P3.4, P8.1 | Offline install, exit-3 diagnostics |
| AC21 performance (§22) | P6.6 | Benchmark job |
| Label composition (§13.10) | P3.3, P5.2, P5.4 | Label goldens |
| Config discovery and roots (§13.2) | P2.2 | Resolver tests |
| Footer grammar (§13.2) | P2.6 | Grammar and property tests |
| Logo sanitizer (§16.5) | P4.4, P4.5 | Allow/reject/canonicalization corpus |
| Generic icons (§13.8) | P4.6 | Hash and embedding tests |
| YAML dialect (§16.4) | P2.1 | Loader tests |
| `SOURCE_DATE_EPOCH` (§16.1) | P3.6 | Usage-error tests |

---

## 17. First implementation sequence

After P0 decisions are accepted, the first reviewable sequence SHOULD be:

1. bootstrap package and verification command (P1.1),
2. schema conventions and resource loading (P1.2),
3. configuration, model, View, theme, manifest, diagnostics and pin schemas (P1.3–P1.6),
4. built-in resources, D2 installer helper, theme hash (P1.7–P1.9),
5. safe YAML loading and project resolution (P2.1, P2.2),
6. domain objects and diagnostics (P2.3, P2.4),
7. semantic validation and footer parser (P2.5, P2.6),
8. selection engine (P2.7),
9. `flowframe validate` and deterministic `review` (P2.8, P2.9),
10. architecture IR, projection and D2 writer (P3.1–P3.3),
11. D2 wrapper, body SVG verification, manifest (P3.4–P3.6),
12. `compile`, `render` and recoverable `build` (P3.7, P3.8),
13. publish v0.1-alpha.1 (P3.9),
14. theme, contrast, sanitizer, icons, text, legend, accessibility and compositor (P4),
15. flow and sequence families (P5),
16. harden, evaluate the AI adapter, package and release (P6–P8).

This order exposes contract and external-renderer mistakes early while keeping branding and all later families on the same validated compiler foundation.
