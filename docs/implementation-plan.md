# FlowFrame — Executable Implementation Plan

## 1. Purpose

This plan decomposes [`prd.md`](prd.md) and [`technical-spec.md`](technical-spec.md) into ordered, independently verifiable work packages for FlowFrame v0.1.

The plan uses completion gates instead of calendar estimates. A phase is complete only when its exit criteria pass in CI. Later work MAY begin in parallel where dependencies allow, but no milestone may be declared complete by deferring a failed gate.

The PRD is authoritative for product behavior. The technical specification is authoritative for the proposed implementation unless an accepted ADR changes it.

---

## 2. Delivery principles

1. **Contracts before behavior.** Examples and tests are written against versioned schemas before compiler features depend on them.
2. **One vertical slice early.** Architecture/infrastructure proves the complete path before adding other diagram families.
3. **Deterministic core before AI.** AI adapters consume the same public contracts and cannot bypass validation.
4. **Presentation is global.** Theme, colors, logo and decoration placement are project-wide, never per-view overrides.
5. **Fail closed at boundaries.** YAML, filesystem paths, SVG/CSS and renderer processes are treated as untrusted.
6. **Every phase leaves a usable repository.** Main must pass formatting, types and the tests available at that phase.
7. **External upgrades are explicit.** D2, schemas, sanitizer profiles and golden snapshots change only through reviewed upgrades.

---

## 3. Milestone map

| Plan phase | PRD stage | Result |
|---|---|---|
| P0 — Decisions and spikes | Stage 0 | External risks resolved and ADRs accepted |
| P1 — Repository and contracts | Stage 1 | Installable CLI shell and validated public schemas |
| P2 — Loading and semantic core | Stage 1 | Safe, location-aware validation with stable diagnostics |
| P3 — Architecture vertical slice | Stage 1 | First complete deterministic build |
| P4 — Design system and presentation | Stage 2 | Global branding, decorations and safe logos |
| P5 — Flow and sequence families | Stage 3 | All MVP families from reusable models |
| P6 — Hardening and release quality | Stages 2, 3 and 5 | Offline, secure and reproducible release candidate |
| P7 — AI adapter and evaluation | Stage 4 | One bounded, contract-driven AI workflow |
| P8 — Packaging and v0.1 release | Stage 5 | Installable, documented v0.1 |
| P9 — Optional capabilities | Stage 6 | Post-MVP work only |

### 3.1. Dependency chain

```text
P0
└── P1
    └── P2
        └── P3
            ├── P4 ──┐
            └── P5 ──┴── P6
                          └── P7
                              └── P8

P9 starts only after v0.1 unless a separate product decision changes scope.
```

P4 theme schema work can start after P1, but SVG composition integrates only after P3 produces body SVG. Flow and sequence IR design can start after P2, but their release gate depends on the deterministic generator and renderer established in P3.

---

## 4. Repository-wide definition of done

Every work item is done only when:

- production code is typed and passes Ruff and mypy at the current project strictness,
- public behavior has unit or integration coverage,
- negative and boundary behavior is tested, not only the happy path,
- new diagnostics have stable codes and source-location assertions,
- new public YAML/JSON structures have schema fixtures,
- output-affecting changes have reviewed D2 or normalized SVG golden updates,
- security-sensitive code has explicit rejection tests,
- documentation and examples match the implemented command names,
- tests require no network access,
- `git diff --check` and the project verification command pass.

Any intentional deviation from the PRD or technical specification requires an ADR and a documentation update in the same change.

---

## 5. P0 — Decisions and technical spikes

### Objective

Remove uncertainty around D2, deterministic SVG generation, text measurement, logo safety and distribution before building the production architecture.

### P0.1 — Establish ADR framework

Deliverables:

- `docs/adr/README.md` with ADR states and template,
- ADR for Python CLI and process-boundary integration with D2,
- ADR for ELK baseline and optional TALA,
- ADR for schema authority and contract versioning,
- ADR for global presentation composition outside D2 body layout.

Acceptance:

- every open Stage 0 choice has an owner ADR or a recorded planned experiment,
- superseded decisions remain discoverable rather than being overwritten.

### P0.2 — Pin and exercise D2

Tasks:

- select a D2 version available for supported macOS and Linux architectures,
- record official artifact URLs, licenses and SHA-256 checksums,
- write a temporary installation script used only by the spike,
- render one architecture, one integration-flow and one sequence fixture with ELK,
- capture reported D2 and layout engine version behavior,
- test `d2 validate` and representative failure modes,
- test invocation with network unavailable and a minimal environment,
- measure startup, render time and output size for small and medium fixtures,
- document flags that affect generated SVG.

Acceptance:

- all three fixtures render offline,
- the chosen executable can be checksum-verified,
- the wrapper can distinguish validation failure, renderer failure and timeout,
- version/flag data needed for the manifest is obtainable or its absence is explicitly modeled.

### P0.3 — Layout and SVG stability spike

Tasks:

- render each fixture repeatedly in two isolated directories,
- compare D2 bytes, raw SVG and normalized SVG,
- inventory nondeterministic SVG fields and ordering,
- exercise long labels, Unicode, nested containers and both relation directions,
- confirm sequence output has sufficient stable hooks for later composition,
- define the minimum SVG normalization behavior without changing painter order,
- record baseline visual snapshots.

Acceptance:

- sources of nondeterminism are understood,
- a normalization approach is selected,
- unsupported D2 constructs are listed,
- no required MVP view needs manual coordinates.

### P0.4 — Presentation composition spike

Create one body SVG and exercise:

- title only,
- title plus subtitle,
- static footer with no source placeholder,
- footer with `{source}`,
- no footer when `{source}` is required but source is absent,
- logo in each supported logo slot,
- title block and logo in distinct top slots,
- wider decoration than body,
- long title and Unicode source,
- slot conflict rejection.

Tasks:

- compare candidate deterministic text-measurement strategies,
- test bundled fonts without host font discovery,
- derive top/bottom band and width expansion calculations,
- verify that translating the body preserves internal geometry,
- verify view-box and canvas behavior in target documentation renderers.

Acceptance:

- an ADR selects the measurement method and supported font formats,
- the band algorithm never clips or overlays the body in the fixture set,
- decoration placement is deterministic without browser automation,
- the same algorithm works on SVG emitted for every MVP family.

### P0.5 — Logo sanitizer spike

Build a representative asset corpus containing:

- simple paths and groups,
- linear and radial gradients,
- clip paths and masks,
- internal `use` and `url(#id)` references,
- every proposed bounded filter primitive,
- a `style` block with allowed CSS,
- scripts and event attributes,
- external references and CSS imports,
- `foreignObject`, animation and raster images,
- excessive nesting, elements, path data and filter regions,
- duplicate or adversarial IDs and namespaces.

Tasks:

- write the exact `flowframe-svg-logo/v1` matrix in `rules/svg-logo-profile-v1.md`,
- validate candidate XML and CSS libraries,
- prototype deterministic ID rewriting and reference normalization,
- select canonical number and namespace serialization,
- choose numeric security limits from measured fixtures,
- confirm full-asset rejection with actionable diagnostics.

Acceptance:

- all allowed fixtures preserve expected appearance,
- all forbidden fixtures fail closed,
- repeat sanitization is byte-identical,
- original and sanitized hashes are stable,
- no regular-expression-only CSS parsing is used.

### P0.6 — Close release decisions

Decide and record:

- package manager/build backend,
- schema compatibility and deprecation policy,
- supported documentation renderers and browsers,
- committed-versus-generated SVG policy,
- icon set and license,
- initial numeric input/process limits,
- baseline performance budgets,
- scope of v0.1 custom fonts.

### P0 exit gate

- all required ADRs are accepted,
- spike code or fixtures are retained only when useful and clearly marked,
- the three representative diagrams render offline,
- presentation and sanitizer designs are proven on real D2 SVG,
- no license, platform or dependency issue blocks P1.

---

## 6. P1 — Repository foundation and public contracts

### Objective

Create an installable, testable Python project with frozen v1 schema drafts and packaged built-in resources.

### P1.1 — Bootstrap the project

Tasks:

- create `pyproject.toml` with Python 3.12 floor and Hatchling build,
- configure `uv.lock`, Ruff, mypy and pytest,
- add `src/flowframe` package and `flowframe` console entry point,
- add version reporting from installed package metadata,
- establish `tests/unit`, `tests/contract`, `tests/integration`, `tests/golden` and `tests/security`,
- add one verification command used locally and in CI,
- ensure wheel and source distribution include schemas and built-in theme resources.

Acceptance:

- a clean environment installs the local package,
- `flowframe version` succeeds,
- an installed wheel can locate all packaged resources without repository-relative paths.

### P1.2 — Define schema conventions

Tasks:

- select JSON Schema Draft 2020-12,
- define stable `$id` naming and local reference conventions,
- require `schemaVersion` discriminators,
- use closed objects unless a specific extension point exists,
- define ID, label, URI-free local path and bounded string primitives,
- add a schema compilation test that resolves every reference offline,
- add a version registry in code.

Acceptance:

- every schema loads and resolves with the network disabled,
- unsupported versions produce one stable diagnostic,
- duplicate or conflicting `$id` values fail tests.

### P1.3 — Project Configuration schema

Implement `flowframe-config.schema.json` for `flowframe-config/v1`:

- required theme ID or versioned built-in default behavior,
- logo asset selection and `top-left`/`top-right` placement,
- title `top-left`/`top-center` placement,
- footer enablement, template and bottom placement,
- no per-view color, font or asset overrides.

Fixtures cover:

- minimal config,
- complete branding config,
- static footer,
- source footer,
- invalid positions,
- same top slot assigned to title and logo,
- unknown properties.

### P1.4 — System Model schema

Tasks:

- encode element, boundary, relation and scenario structures,
- encode the controlled vocabularies from the PRD,
- require `source` and `target` for directed and undirected relations,
- restrict scenario note `participant` syntactically to an element-ID-shaped reference,
- prohibit visual styling fields,
- define limits for labels, descriptions and IDs.

Fixtures include every kind and semantic value plus intentionally invalid styling and reference shapes.

### P1.5 — View Specification schema

Tasks:

- encode family/subtype combinations,
- encode audience, detail, selection and portable layout preferences,
- encode family-specific required fields,
- restrict `presentation` to `title`, `subtitle` and `source`,
- reject theme, color, font, logo and position overrides,
- encode documented presentation text limits.

### P1.6 — Theme and manifest schemas

Theme schema includes:

- `flowframe-theme/v1` metadata,
- token definitions and semantic mappings,
- typography and decoration tokens,
- declared assets with media type, alternative text, path and display bounds,
- license metadata,
- raw colors only in theme token values.

Manifest schema includes:

- `flowframe-manifest/v1`,
- configuration/model/view paths, schema versions and hashes,
- resolved theme ID/version/origin/hash,
- renderer/layout versions and flags,
- sanitizer profile and implementation identity,
- per-asset original and sanitized hashes,
- D2/SVG output hashes,
- `generatedAt`.

### P1.7 — Built-in resources and examples

Tasks:

- create immutable `flowframe-light` resource pack,
- include an explicit package resource manifest,
- add license inventory,
- create minimum valid project, model and View fixtures,
- add one custom project theme fixture but do not yet render it.

### P1 exit gate

- all five schemas pass positive and negative fixture suites,
- schemas and built-in theme are present in an installed wheel,
- unknown properties and per-view style fields are rejected,
- examples parse as YAML and validate structurally,
- the CLI shell and project verification command pass on macOS and Linux CI.

---

## 7. P2 — Safe loading, domain model and semantic validation

### Objective

Turn structurally valid source files into immutable, location-aware domain objects and reject every invalid cross-document state before projection.

### P2.1 — Safe YAML loader

Tasks:

- enforce UTF-8 policy and input byte limits,
- use safe construction only,
- reject duplicate keys,
- bound nesting and aliases,
- convert values to JSON-compatible primitives,
- build JSON Pointer to file/line/column mapping,
- test malformed YAML and resource-exhaustion cases.

### P2.2 — Project and path resolver

Tasks:

- implement explicit `--config` resolution,
- implement nearest-parent `flowframe.yaml` discovery,
- implement the versioned built-in config when absent,
- resolve project root from config location,
- resolve themes project-first then package resources,
- reject traversal and symlink escape,
- expose logical relative paths separately from physical paths.

Acceptance:

- process working directory does not change resolution,
- two nested configs resolve to the nearest one,
- absent config cannot access project theme directories,
- the chosen config and theme origin are observable for manifest construction.

### P2.3 — Domain object construction

Create frozen typed models for:

- project configuration and branding slots,
- theme tokens, mappings and assets,
- system elements, boundaries, relations and scenarios,
- view selection, presentation and family settings,
- source references and normalized IDs.

Constructors receive only schema-valid primitive values. Domain objects contain no YAML-library nodes.

### P2.4 — Diagnostics infrastructure

Tasks:

- implement severity, code, message, primary location, related locations and hint,
- define deterministic sort order,
- implement text and JSON renderers,
- map JSON Schema paths back to YAML locations,
- reserve diagnostic prefixes from the technical specification,
- add redaction rules for absolute paths and exception details.

### P2.5 — Semantic validator

Implement and test:

- global ID uniqueness in each namespace,
- element/boundary/relation/scenario reference resolution,
- boundary containment and cycle detection,
- valid parent types and family constraints,
- both endpoints required for undirected relations,
- relation semantic/directionality compatibility,
- scenario participant resolution,
- note participant restricted to elements, never boundaries,
- scenario message references and order,
- theme token existence and semantic mapping coverage,
- asset declaration and media-type consistency,
- branding slot conflicts,
- footer grammar and escaped brace rules.

### P2.6 — Footer template parser

Implement parser tokens, not string replacement:

- literals,
- one optional source placeholder,
- escaped opening and closing braces,
- unmatched-brace and unknown-placeholder diagnostics,
- source inserted once and never reparsed.

Property and fuzz tests cover arbitrary brace sequences and Unicode.

### P2.7 — `validate` command

Tasks:

- connect resolver, loaders, schemas and semantic validators,
- support human and JSON diagnostics,
- return exit code 0 or 1 as appropriate,
- define deterministic diagnostic order across documents,
- ensure validation performs no rendering and writes no artifacts.

### P2 exit gate

- every invalid reference and containment cycle in the corpus has an actionable location,
- footer grammar behavior exactly matches the PRD,
- path resolution is independent of working directory,
- repeated validation emits diagnostics in identical order,
- `flowframe validate` has CLI contract tests for success and failure.

---

## 8. P3 — Architecture vertical slice

### Objective

Produce `diagram.d2`, `diagram.svg` and `manifest.json` transactionally from one infrastructure view with the built-in theme and ELK.

### P3.1 — Deterministic selection engine

Tasks:

- implement explicit includes and documented filters,
- implement excludes and expansion rules,
- derive required containing boundaries,
- include relations only for eligible endpoints,
- retain inclusion reasons for debugging,
- enforce stable source order with ID tie-breakers,
- warn or error on empty selections according to PRD semantics.

### P3.2 — Architecture IR

Define immutable IR types for:

- node identity, label and semantic role,
- nested boundary groups,
- typed edge with directionality and labels,
- portable layout hints,
- stable ordering.

IR validation tests cover duplicates, dangling references, invalid nesting and unsupported properties.

### P3.3 — Infrastructure projector

Tasks:

- map selected model elements to architecture nodes,
- map boundaries without duplicating elements,
- map relation semantics and directionality,
- preserve source references for diagnostics,
- exclude all visual token values from the IR,
- add focused projection golden fixtures.

### P3.4 — D2 writer and architecture generator

Tasks:

- implement audited string/identifier escaping,
- implement fixed property ordering and one-newline serialization,
- emit generated-file header without volatile values,
- map semantic roles through built-in theme tokens/classes,
- emit nodes before relations,
- prohibit raw D2 from source documents,
- add byte-identical D2 golden tests.

### P3.5 — D2 process wrapper

Tasks:

- locate and verify pinned D2,
- invoke without a shell,
- use explicit ELK selection and cleaned environment,
- enforce timeout and output limits,
- capture bounded stderr for diagnostics,
- validate before render,
- reject partial output on non-zero exit.

### P3.6 — Body SVG verification

Tasks:

- parse with hardened XML settings,
- reject external references and unsafe nodes,
- verify a finite view box and canvas bounds,
- normalize only fields approved by the Stage 0 ADR,
- expose body bounds for later presentation composition.

### P3.7 — Manifest collector and writer

Tasks:

- record source/config/theme hashes and schema versions,
- record D2/layout identity and flags,
- record output hashes,
- serialize stable JSON and validate it against manifest schema,
- ensure `generatedAt` never affects D2 or SVG.

### P3.8 — Transactional `compile` and `build`

Tasks:

- write into a sibling staging directory,
- publish all three files only on complete success,
- preserve the previous successful build on failure,
- return specified exit categories,
- add interruption and stale-partial-output tests.

### P3.9 — Alpha.1 example

Add the payments example required by the PRD:

```text
examples/payments/flowframe.yaml
examples/payments/system-model.yaml
examples/payments/infrastructure-view.yaml
```

One documented command creates:

```text
build/payments/infrastructure/
├── diagram.d2
├── diagram.svg
└── manifest.json
```

### P3 exit gate — v0.1-alpha.1

- a clean offline environment builds the payments infrastructure view,
- repeat runs produce byte-identical D2 and normalized body SVG,
- manifest validates and contains the pinned toolchain identity,
- renderer failures preserve the previous complete output,
- no source document can inject D2 or visual style,
- all available CI platforms pass.

---

## 9. P4 — Design system, branding and presentation

### Objective

Apply one project-wide Theme/Brand Pack and safely add title, subtitle, source footer and logo without modifying semantic layout.

### P4.1 — Complete design tokens and mappings

Tasks:

- implement every PRD token class,
- implement mappings from element kinds and relation semantics,
- provide documented built-in fallbacks for optional mappings,
- reject unknown token references,
- verify meaning remains readable in grayscale,
- add contrast checks for essential text and strokes.

### P4.2 — Theme resolver and aggregate hash

Tasks:

- load exactly one complete theme pack,
- prohibit partial merging of project and built-in packs,
- resolve only missing semantic mappings from the versioned built-in theme and require their resulting tokens in the selected theme,
- resolve only declared local assets,
- hash canonical theme metadata and referenced assets in stable path order,
- record `project` or `built-in` origin,
- ensure project themes are rooted at `<project-root>/themes/<id>/`.

### P4.3 — Normative sanitizer profile

Finalize `rules/svg-logo-profile-v1.md` with:

- exact allowed elements,
- allowed namespaced and presentation attributes,
- accepted CSS selectors/properties/value grammar,
- gradient, clip, mask and internal-reference rules,
- bounded filter primitive and attribute matrix,
- numeric global limits,
- explicit rejection matrix,
- canonicalization and ID-rewrite algorithm,
- conformance examples.

The document and implementation share profile fixtures; neither silently broadens the other.

### P4.4 — Production logo sanitizer

Tasks:

- enforce raw byte cap before parsing,
- parse XML with DTD/entities/network disabled,
- enforce element, nesting, path and filter-region limits,
- parse CSS with `tinycss2`,
- validate all reference graphs and reject missing/cyclic invalid references,
- reject external or data URLs and raster images,
- fail on unknown content without stripping it,
- rewrite IDs and references deterministically,
- produce canonical bytes and both hashes,
- require and expose finite intrinsic bounds to the compositor.

Acceptance:

- the complete P0 corpus passes expected allow/reject results,
- sanitizer results are byte-identical across repeated supported-platform runs,
- manifest records profile, implementation/version and both hashes.

### P4.5 — Text and footer fragments

Tasks:

- use only pinned theme fonts and the chosen deterministic metric strategy,
- render title/subtitle as a single measurable block,
- render static and source footer tokens as escaped plain text,
- omit a source-dependent footer when source is absent,
- preserve a static footer when source is absent,
- enforce presentation text limits before rendering.

### P4.6 — Decoration compositor

Implement in this order:

1. read verified body bounds,
2. measure title block, logo and footer,
3. fit logo within per-asset display bounds with aspect ratio preserved,
4. calculate top and bottom bands using `decoration-gap`,
5. expand canvas width for wider decorations,
6. translate body once,
7. place fragments in supported slots using collision-safe deterministic IDs,
8. normalize and verify final SVG.

Tests cover:

- no decorations,
- every individual decoration,
- all valid slot combinations,
- invalid same-slot title/logo combination,
- absent optional metadata,
- long and Unicode text,
- logo wider/taller than body before fitting,
- logo IDs colliding with body IDs,
- deterministic title/logo accessibility metadata,
- narrow and wide bodies,
- body and decorations at canvas security limits.

### P4.7 — Global consistency enforcement

Tasks:

- reject per-view theme, raw style, font, logo and position fields,
- build all three family fixtures with one project theme when available,
- prove every output resolves the same theme hash,
- add a custom branded fixture with logo, title and source footer,
- ensure config changes affect all project diagrams on rebuild.

### P4.8 — Manifest presentation provenance

Record:

- resolved config path or built-in ID/version,
- theme identity, origin and aggregate hash,
- sanitizer profile and implementation identity,
- used asset input and sanitized hashes,
- final decorated SVG hash.

### P4 exit gate

- built-in and custom theme fixtures render deterministically,
- logo/title/footer never cover or clip the body,
- static and `{source}` footer semantics match the PRD,
- every unsafe/over-complex logo fixture is rejected,
- presentation is identical in policy across available diagram families,
- manifest contains sufficient information to reproduce asset processing.

---

## 10. P5 — Integration-flow and sequence families

### Objective

Complete the three-family MVP while reusing the same System Model, validation, theme and output pipeline.

### P5.1 — Flow IR and projector

Tasks:

- define directed flow nodes and edges,
- map integration relations, protocols and data/event annotations,
- validate unsupported directionality or semantics,
- implement family-specific selection expansion,
- add small, medium and nested-boundary fixtures,
- add deterministic projection and D2 goldens.

### P5.2 — Flow generator

Tasks:

- emit flow-specific D2 constructs through the shared writer,
- map semantics to global tokens/classes,
- verify arrow/label accessibility without color-only meaning,
- render and snapshot with ELK.

### P5.3 — Sequence IR and projector

Tasks:

- derive ordered participants from a selected scenario,
- implement messages and note steps in source order,
- require every note participant to resolve to an element,
- validate relation-backed messages when used,
- keep boundary IDs out of participant references,
- preserve explicit ordering independent of model map order.

### P5.4 — Sequence generator

Tasks:

- emit supported D2 sequence constructs,
- map message kinds and directionality consistently,
- render notes safely,
- cover repeated participants, self-message if supported and long labels,
- document any D2 limitations accepted by the Stage 0 ADR.

### P5.5 — Cross-family reuse example

Extend the payments model with:

- one integration-flow view,
- one sequence scenario and view,
- no duplicated system inventory,
- shared global theme and branding.

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

The corpus includes:

- five infrastructure views,
- five integration-flow views,
- five sequence views,
- small, medium and boundary-size cases,
- nested boundaries,
- long and Unicode labels,
- missing optional icons,
- omitted presentation metadata,
- custom global theme with every decoration,
- deliberately invalid contracts and semantic states.

### P6.2 — Reproducibility

Tasks:

- run identical builds in separate absolute paths,
- run twice on each supported platform,
- compare D2 bytes,
- compare normalized SVG bytes within the declared toolchain equivalence class,
- compare stable manifest fields,
- verify no absolute paths, random IDs or locale-dependent values leak,
- set locale/timezone explicitly in CI where needed.

### P6.3 — Security suite

Tasks:

- YAML alias/nesting/input-size attacks,
- path traversal and symlink escape,
- malicious labels aimed at D2 injection,
- XML entities and DTDs,
- SVG scripts, events, external URLs and CSS imports,
- excessive SVG/filter/path complexity,
- renderer timeout and output flooding,
- unsafe environment influence,
- stale/partial artifact publication.

### P6.4 — Accessibility and visual review

Tasks:

- automate contrast checks where geometry permits,
- generate grayscale snapshots,
- assert semantic distinctions do not rely only on color,
- conduct human review of the full corpus,
- record accepted renderer quirks,
- require explicit review for all golden image changes.

### P6.5 — CLI contract completion

Tasks:

- test every command help page,
- test exit codes 0–5,
- test text and JSON diagnostics,
- provide concise success output,
- hide stack traces outside debug mode,
- ensure interrupted builds return non-success and keep previous artifacts.

### P6.6 — Performance gates

Tasks:

- benchmark small and medium corpus builds,
- separate Python compilation from D2 render time,
- set regression thresholds from P0 baselines,
- enforce global time/memory/output bounds,
- document that large unreadable diagrams should be split by views.

### P6 exit gate — release candidate

- all PRD acceptance criteria unrelated to AI and packaging pass,
- the full suite runs offline,
- supported-platform reproducibility is documented and tested,
- security corpus fails closed,
- full visual review has no unresolved blocker,
- performance stays within accepted budgets.

---

## 12. P7 — AI adapter and evaluation

### Objective

Add one optional adapter that produces public structured contracts without receiving control of styling or deterministic output.

### P7.1 — Adapter boundary

Tasks:

- define provider-neutral request/result types,
- separate source-document extraction from model/view generation,
- accept structured output only,
- keep credentials and network logic outside the compiler,
- make the package usable without AI dependencies installed.

### P7.2 — Generation prompts and rules

Create:

- System Model generation prompt,
- View Specification generation prompt,
- explicit uncertainty/assumption format,
- rule forbidding raw styles and invented source claims,
- instructions to reuse stable IDs and existing model content,
- source-conformance mode.

### P7.3 — Validation-repair loop

Tasks:

- validate generated YAML through the normal pipeline,
- return structured diagnostics to the adapter,
- bound repair attempts,
- prevent repairs from editing project config or Theme/Brand Pack unless explicitly authorized,
- show the final source diff for human review.

### P7.4 — Evaluation corpus

Measure:

- schema-valid first-pass rate,
- semantic-valid rate after bounded repair,
- expected element/relation/scenario fact recall,
- unsupported-fact or hallucination rate,
- stability of IDs across repeat requests,
- absence of styling fields,
- correct view family and selection.

Set the v0.1 threshold from P0/P7 baseline before release acceptance.

### P7 exit gate

- one adapter generates models and views that enter the unchanged compiler,
- invalid AI output cannot bypass validation,
- the evaluation threshold is defined and achieved,
- raw theme/style values are never generated,
- assumptions and source gaps remain visible to the user.

---

## 13. P8 — Packaging, CI and v0.1 release

### Objective

Ship a verifiable package and a documented offline workflow.

### P8.1 — Distribution

Tasks:

- build wheel and source distribution,
- bundle schemas, built-in theme, fonts and required licenses,
- provide pinned D2 installation with checksum verification or a documented external prerequisite,
- verify package resource access outside the source checkout,
- generate a software/dependency provenance inventory.

### P8.2 — CI matrix

Required jobs:

- formatting/linting,
- static typing,
- unit and contract tests,
- integration tests with pinned D2,
- security tests,
- deterministic/golden tests,
- package build and installed-wheel smoke test,
- license/provenance check.

At least macOS and Linux run the installed-wheel smoke test. Expensive visual suites MAY run once per pinned renderer platform if cross-platform equivalence has been demonstrated.

### P8.3 — Consumer documentation

Document:

- installation of FlowFrame and pinned D2,
- project structure,
- creating `flowframe.yaml` and a global theme,
- creating model and view files,
- building one or all diagrams,
- title, subtitle, static/source footer and logo behavior,
- asset and font restrictions,
- output/manifest interpretation,
- CI examples for GitHub Actions and GitLab CI,
- upgrades and golden snapshot review,
- troubleshooting by diagnostic code.

### P8.4 — Release checklist

- all PRD MVP acceptance criteria pass,
- schemas and CLI help are versioned,
- package contains required assets and licenses,
- release artifacts have checksums,
- changelog and known limitations are published,
- installation is tested from the release artifact,
- no open critical security or data-loss issue remains,
- exact D2/sanitizer/theme versions are recorded.

### P8 exit gate — v0.1

A new consumer repository can install FlowFrame, create a project-wide branded configuration and produce architecture, flow and sequence SVGs offline from validated YAML using documented commands.

---

## 14. P9 — Post-MVP backlog

These items do not block v0.1:

- optional TALA adapter and comparison workflows,
- dark built-in theme,
- additional architecture and flow subtypes,
- vendor icon packs with independent licenses,
- PNG, PDF and PPTX outputs,
- Windows release support,
- schema migration tooling,
- advanced visual-analysis loop,
- public extension/plugin API,
- performance cache with complete content-addressed keys.

Every post-MVP item requires its own contract and security review. TALA MUST remain opt-in and MUST never become an implicit fallback.

---

## 15. Recommended issue breakdown

Create one tracking epic per phase and one issue per numbered work package. Each issue SHOULD contain:

```text
Context:
  Links to exact PRD and technical-spec sections.

Scope:
  Concrete implementation and files affected.

Out of scope:
  Explicitly deferred behavior.

Acceptance:
  Observable checks, fixtures and commands.

Dependencies:
  Blocking issue IDs and accepted ADRs.

Security/reproducibility:
  Boundary conditions introduced by the change.

Artifacts:
  Schema, code, fixtures, diagnostics and documentation expected.
```

Do not group unrelated schema, renderer and presentation changes into one review. Small contract-first changes keep generated diffs and security decisions reviewable.

---

## 16. Traceability matrix

| PRD requirement | Primary implementation packages | Main verification |
|---|---|---|
| Versioned YAML contracts | `contracts`, `domain`, `validation/schema.py` | Schema fixture suite |
| Semantic correctness | `validation/semantic.py` | Invalid-model corpus |
| Deterministic selection | `selection` | Unit and property tests |
| Family-specific behavior | `projection`, `ir` | Projection goldens |
| Deterministic D2 | `generation` | Byte golden tests |
| ELK/D2 rendering | `rendering/d2_process.py` | Offline integration tests |
| Global theme | `presentation/theme_resolver.py` | Cross-view/family consistency tests |
| Title/subtitle/footer/logo | `presentation` | Decorated SVG snapshots |
| Footer grammar | `presentation/footer_template.py` | Grammar and fuzz tests |
| Safe deterministic logos | `presentation/logo_sanitizer.py` | Allow/reject/canonicalization corpus |
| No decoration overlap | `presentation/compositor.py` | Bounds assertions and visual review |
| Provenance | `manifest` | Schema and hash assertions |
| Secure boundaries | loaders, resolver, sanitizer, process wrapper | Security suite |
| AI isolation | optional adapter | Evaluation and bypass tests |
| Offline operation | complete runtime | Network-disabled CI job |

---

## 17. First implementation sequence

After P0 decisions are accepted, the first reviewable sequence SHOULD be:

1. bootstrap package and verification command,
2. add schema conventions and resource loading,
3. add Project Configuration and built-in theme schemas,
4. add System Model and View schemas,
5. add safe YAML loading and location mapping,
6. add immutable domain objects and diagnostics,
7. add semantic indexes and validators,
8. add `flowframe validate`,
9. add architecture selection, IR and projection,
10. add deterministic D2 writer,
11. add pinned D2 wrapper and body SVG verification,
12. add transactional build and manifest,
13. publish v0.1-alpha.1,
14. add production theme, sanitizer and decoration compositor,
15. add flow and sequence families,
16. harden, evaluate AI adapter, package and release.

This order exposes contract and external-renderer mistakes early while keeping branding and all later families on the same validated compiler foundation.
