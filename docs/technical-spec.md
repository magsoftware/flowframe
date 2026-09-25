# FlowFrame — Technical Design Specification

## 1. Status and authority

| Field | Value |
|---|---|
| Target | FlowFrame v0.1 |
| Status | Draft for implementation |
| Runtime | Python 3.12+ |
| Primary renderer | D2 CLI |
| Default layout | ELK |
| Primary output | SVG |

This document turns the requirements in [`prd.md`](prd.md) into an implementation design. The PRD remains authoritative for product behavior and public contracts. If the documents disagree, implementation MUST follow the PRD and this specification MUST be corrected.

The keywords **MUST**, **SHOULD** and **MAY** have the same meaning as in the PRD.

---

## 2. Design goals

The implementation MUST:

- keep Project Configuration, System Model, View Specification, Theme/Brand Pack and manifest as versioned public contracts,
- separate parsing, validation, projection, code generation, rendering and SVG composition,
- produce byte-identical D2 for identical normalized inputs and compiler version,
- produce reproducible SVG when the complete pinned rendering toolchain is identical,
- work without network access after installation,
- reject unsafe or ambiguous inputs before invoking D2,
- apply one global Theme/Brand Pack to every view in a project,
- keep AI outside the deterministic compiler core,
- make failures actionable through stable diagnostic codes and source locations.

The v0.1 implementation does not provide a GUI, manual placement, arbitrary per-view styling, general-purpose SVG sanitization or a public plugin API.

---

## 3. Selected implementation stack

| Concern | Selection | Notes |
|---|---|---|
| Language | Python 3.12+ | Type annotations are mandatory in production code. |
| Packaging | `pyproject.toml` with `uv` | Lock application and development dependencies. Build wheels with Hatchling. |
| CLI | Typer | Thin command layer; business logic stays callable without a terminal. |
| YAML | `ruamel.yaml` parser/composer | Retain node marks, allow only core YAML tags and construct JSON-compatible values in FlowFrame code. |
| Schema validation | `jsonschema`, Draft 2020-12 | JSON Schema files are the public structural contracts. |
| Internal models | frozen dataclasses and enums | Constructed only after structural validation; avoid a second public validation contract. |
| XML/SVG | hardened `lxml` parser | Disable entity resolution, DTD loading and network access. |
| CSS parsing | `tinycss2` | Required for allowlist validation of SVG `<style>` content. |
| Hashing | Python `hashlib.sha256` | Hash raw source bytes and canonical processed bytes as defined below. |
| D2 integration | pinned D2 CLI through `subprocess` | No shell invocation. Direct Go integration is deferred. |
| Tests | `pytest` | Unit, contract, integration, golden, snapshot and security suites. |
| Static quality | Ruff and mypy | Formatting/linting and strict checks for the core packages. |

Dependency versions MUST be locked for releases. D2 itself MUST be installed at the version selected by the Stage 0 ADR and verified by checksum. The repository MUST NOT silently use an arbitrary executable found on `PATH` in reproducible or CI mode.

### 3.1. Supported platforms

v0.1 targets macOS and Linux on architectures for which the pinned D2 release is available. Windows MAY work through Python and D2 but is not a release gate until a Windows CI job is explicitly added.

---

## 4. Runtime architecture

```text
CLI / Python facade
        |
        v
project resolver ----> package resources
        |               (schemas, built-in theme)
        v
safe loaders -> schema validation -> semantic validation
        |                                      |
        +-------------- diagnostics <---------+
        |
        v
view selector -> family projector -> typed family IR
        |                                  |
        |                                  v
        |                         deterministic D2 emitter
        |                                  |
        |                                  v
        |                         D2 validate + render
        |                                  |
        +---------- theme/assets ----------+
                                           v
                               SVG decoration compositor
                                           |
                                           v
                              normalize -> verify -> manifest
```

Each arrow is a typed module boundary. A module MUST NOT reach around the boundary to read source YAML again. In particular:

- project resolution owns filesystem discovery,
- loaders own source text and location maps,
- validators own all user-facing input rejection,
- selectors and projectors consume validated internal models,
- emitters consume only an IR plus a resolved theme,
- the renderer consumes generated D2 and explicit renderer options,
- the compositor consumes renderer SVG plus measured decoration fragments,
- the manifest builder receives recorded facts from each stage rather than rediscovering them.

---

## 5. Proposed package layout

```text
src/flowframe/
├── cli.py
├── api.py
├── diagnostics.py
├── errors.py
├── project/
│   ├── resolver.py
│   ├── paths.py
│   └── resources.py
├── contracts/
│   ├── loader.py
│   ├── locations.py
│   └── versions.py
├── domain/
│   ├── config.py
│   ├── model.py
│   ├── view.py
│   ├── theme.py
│   └── manifest.py
├── validation/
│   ├── schema.py
│   ├── semantic.py
│   ├── presentation.py
│   └── assets.py
├── selection/
│   ├── predicates.py
│   └── selector.py
├── projection/
│   ├── architecture.py
│   ├── flow.py
│   └── sequence.py
├── ir/
│   ├── common.py
│   ├── architecture.py
│   ├── flow.py
│   └── sequence.py
├── generation/
│   ├── d2_writer.py
│   ├── escaping.py
│   └── families/
├── rendering/
│   ├── d2_process.py
│   ├── svg_verify.py
│   └── normalization.py
├── presentation/
│   ├── theme_resolver.py
│   ├── footer_template.py
│   ├── text_fragments.py
│   ├── logo_sanitizer.py
│   └── compositor.py
├── manifest/
│   ├── collector.py
│   └── writer.py
└── resources/
    ├── schemas/
    └── themes/flowframe-light/
```

Public entry points SHOULD be limited to the CLI and a small facade in `api.py`. Internal modules MAY change before v1 without compatibility guarantees.

---

## 6. Public contracts and resolution

### 6.1. Contract files

The canonical schemas are:

- `schema/flowframe-config.schema.json` for `flowframe-config/v1`,
- `schema/system-model.schema.json` for `flowframe/v1` System Models,
- `schema/view.schema.json` for `flowframe/v1` View Specifications,
- `schema/flowframe-theme.schema.json` for `flowframe-theme/v1`,
- `schema/flowframe-manifest.schema.json` for `flowframe-manifest/v1`.

Schemas MUST set `additionalProperties: false` at every closed object boundary. Extensions, if later supported, require a dedicated namespaced field rather than accepting misspelled properties.

### 6.2. Project root

Resolution is deterministic:

1. If `--config` is given, resolve that file and use its parent as project root.
2. Otherwise walk from the requested System Model directory toward the filesystem root for the nearest `flowframe.yaml`.
3. If none exists, use the System Model directory as the effective root and create the versioned built-in configuration in memory.
4. Resolve all project-relative paths against that root, never against the process working directory.

The resolver MUST canonicalize paths, reject traversal outside the project root for project-owned assets and retain both a logical relative path and a resolved filesystem path. Symlinks that escape the project root MUST be rejected for themes and assets.

### 6.3. Theme resolution

For theme ID `X`:

1. look for `<project-root>/themes/X/theme.yaml`,
2. otherwise look for immutable packaged resource `resources/themes/X/theme.yaml`,
3. fail if neither exists.

The first match is final; file-level overlay or merging between a project theme and a built-in theme is forbidden in v0.1. At semantic-mapping resolution only, a missing element-kind or relation-semantic mapping uses the corresponding mapping from the versioned built-in theme. The resulting token reference MUST exist in the selected theme. Missing files, malformed tokens or unknown token references never trigger a switch to another theme.

The manifest MUST record theme ID, declared version, origin (`project` or `built-in`) and an aggregate hash. The aggregate is calculated from a domain tag followed by each normalized POSIX relative path, byte length and raw file bytes for `theme.yaml` and all referenced assets, sorted by path. Length prefixes prevent ambiguous concatenation.

---

## 7. Loading and source locations

The loader pipeline is:

1. read bytes with an explicit UTF-8 policy,
2. reject a byte-order mark only if the contract ADR chooses strict UTF-8; otherwise normalize it consistently,
3. enforce input byte and YAML nesting limits,
4. compose a YAML node graph without invoking custom constructors,
5. allow only the documented core scalar, sequence and mapping tags,
6. reject duplicate keys, aliases exceeding the configured expansion limit and non-string mapping keys where the schema expects objects,
7. construct JSON-compatible primitive values in FlowFrame code,
8. build a JSON Pointer to source-location map,
9. validate against the selected schema,
10. construct immutable domain objects.

The location map stores file, one-based line and column, plus the nearest JSON Pointer. Semantic validation errors MUST refer to the most specific source field available.

Input order MAY be retained for author-friendly output, but semantic equality and generated ordering MUST not depend on YAML parser implementation details.

---

## 8. Validation model

Validation runs in layers and stops only where continuing would create misleading diagnostics:

1. project/configuration schema,
2. theme schema and theme asset inventory,
3. System Model schema,
4. View schema,
5. cross-document semantic validation,
6. selection validation,
7. presentation and asset policy,
8. family IR invariants,
9. generated D2 lint,
10. `d2 validate`,
11. render process result,
12. output SVG safety and integrity checks,
13. manifest schema validation.

Structural errors in one document prevent construction of its domain model. Independent documents MAY still be checked so one invocation can report useful errors together.

### 8.1. Core semantic indexes

Build once per System Model:

- `element_by_id`,
- `boundary_by_id`,
- `relation_by_id`,
- `scenario_by_id`,
- parent-to-children and child-to-parent boundary maps,
- outgoing and incoming relation maps,
- scenario participant and referenced-relation sets.

All IDs are case-sensitive. Duplicate detection occurs before references are resolved. Containment cycles MUST be detected with a deterministic depth-first traversal and reported as the shortest reproducible cycle path available.

For an undirected relation, `source` and `target` remain mandatory. Directionality affects arrowheads and eligible labels only; it does not change the storage model.

Scenario note `participant` references an element, never a boundary.

### 8.2. Diagnostics

A diagnostic contains:

```text
severity: error | warning | info
code: stable FlowFrame code
message: concise human explanation
location: file + line + column + JSON Pointer
related: zero or more related locations
hint: optional corrective action
```

Code families:

| Prefix | Area |
|---|---|
| `FFC` | Project configuration and resolution |
| `FFS` | JSON Schema and source loading |
| `FFM` | System Model semantics |
| `FFV` | View and selection semantics |
| `FFT` | Theme, presentation and assets |
| `FFI` | Family IR invariants |
| `FFD` | D2 generation and validation |
| `FFR` | Renderer and SVG output |
| `FFX` | Internal/unexpected failures |

Codes and exit statuses are public CLI behavior. Message wording MAY improve without a major schema change.

---

## 9. Selection and projection

### 9.1. Selection

Selection MUST be a pure function of validated System Model plus View Specification. It returns:

- selected elements,
- selected boundaries,
- selected relations or scenario,
- deterministic inclusion reasons,
- warnings for empty or unexpectedly disconnected results.

Selection follows the PRD order exactly:

1. Build the seed set from `select.ids`, `select.tags` and `select.kinds`. Values within one field use OR and populated fields combine with AND. If `select` is omitted, all elements are seeds.
2. When `includeRelated` is true, perform breadth-first traversal through relations for exactly the configured `relatedDepth` limit and add opposite endpoints.
3. Apply exclusions; a match in any populated exclusion field removes the element and exclusion wins over selection or traversal.
4. Include a relation only when both endpoints remain selected.
5. Add the complete containment-ancestor chain. Hide or promote boundaries according to `display.boundaries`.
6. Omit empty visible boundaries.

A sequence view resolves its single `scenarioId` and its required participants using the sequence-specific contract rather than silently dropping an excluded participant.

Selection results MUST be ordered by stable source order with ID as a tie-breaker. Sets and hash-map iteration MUST never leak into generated output.

### 9.2. Family projectors

Each projector maps the selection to one of three disjoint IRs:

- Architecture IR: nodes, nested boundary groups, typed edges and portable layout hints.
- Flow IR: steps/nodes, directed semantic flows, labels and protocol/data annotations.
- Sequence IR: ordered participants, messages, notes and grouping constructs.

A projector MUST NOT emit D2 text or inspect theme colors. It MAY assign semantic roles such as `external`, `database` or `async` that the generator resolves through the global theme.

Every IR constructor enforces:

- unique IDs within the IR,
- valid references,
- family-supported relation/message types,
- resolved directionality,
- valid boundary nesting,
- normalized optional text,
- stable item ordering.

---

## 10. Deterministic D2 generation

The D2 generator is a programmatic typed writer, not an unrestricted text-template engine. It MUST:

- write a generated-file header with FlowFrame version,
- emit only constructs owned by the relevant family generator,
- escape IDs, labels and string values in one audited module,
- serialize properties in a fixed order,
- serialize nodes before edges,
- preserve IR order and use stable ID tie-breakers,
- resolve all visual values through theme tokens and framework mappings,
- avoid timestamps, absolute paths and process-specific values,
- finish files with exactly one newline.

Raw D2 snippets from YAML are forbidden. Unknown tokens, kinds or semantics are validation errors rather than pass-through content.

Generated identifiers SHOULD use validated source IDs where D2 permits. Any derived identifier uses a documented prefix and a stable collision suffix calculated from semantic identity, never a random UUID or Python hash.

The generated D2 is an auditable intermediate artifact. Byte-for-byte golden tests are the main regression boundary between projection and external rendering.

---

## 11. D2 process boundary

The renderer wrapper invokes D2 without a shell and with:

- an explicit executable path,
- an explicit layout engine,
- pinned or recorded flags,
- a sanitized environment,
- a bounded working directory inside the build workspace,
- a configurable timeout subject to a global maximum,
- bounded captured stdout and stderr,
- separate validation and render steps where supported.

Potentially influential `D2_*` environment variables MUST be cleared unless explicitly set by FlowFrame. Network access is not required and remote asset references are already rejected before this stage.

The wrapper records executable hash where practical, reported D2 version, layout engine and version if exposed, arguments and normalized failure details. A timeout or non-zero exit is an error even if a partial SVG exists. Partial outputs MUST be removed or left only in a diagnostic temporary directory, never published as successful artifacts.

Build artifacts are first written to a sibling staging directory and moved into the target only after the complete build succeeds. A failed build MUST NOT mix new and previous artifacts.

---

## 12. Global presentation and branding

### 12.1. Ownership model

Visual decisions belong to one resolved Theme/Brand Pack and project configuration. System Model and View Specification MUST NOT contain colors, fonts, line weights, arbitrary shapes, logo paths or decoration positions.

A View MAY provide only content metadata:

```yaml
presentation:
  title: Payments platform
  subtitle: Production infrastructure
  source: Architecture repository
```

Project configuration decides whether and where those values are rendered. Per-view theme or branding overrides are invalid.

### 12.2. Footer template grammar

The footer template supports literal text plus zero or one `{source}` placeholder.

- `{source}` inserts the View source as escaped plain text.
- `{{` produces a literal `{`.
- `}}` produces a literal `}`.
- unmatched braces are errors.
- unknown placeholders are errors.
- a second `{source}` is an error.
- braces in source data are not parsed again.
- a static template is rendered whenever the footer is enabled.
- a template containing `{source}` is omitted when the View has no source.

Parsing produces a small token list (`Literal` and `Source`) during configuration validation. Rendering never performs ad hoc string replacement.

### 12.3. Decoration layout

Decorations are composed after D2 renders the semantic body:

1. parse and verify the body SVG,
2. derive its view box and measurable body bounds,
3. render title and subtitle as one title block using pinned theme fonts,
4. sanitize, canonicalize and measure the selected logo; require a finite intrinsic view box,
5. expand the logo proportionally into its per-asset `maxWidth` × `maxHeight` display box,
6. render the footer fragment,
7. compute the top band as the maximum occupied top-slot height plus `decoration-gap` above and below,
8. compute the bottom band from the footer height plus `decoration-gap` above and below,
9. expand width when any decoration requires it,
10. translate the semantic body without changing its internal geometry,
11. place decorations into their configured slots and embed the sanitized logo nodes,
12. normalize and verify the complete SVG.

The title and subtitle share one primary top slot. A logo and title configured for the same top slot are invalid. The initial allowed slots are:

- logo: `top-left` or `top-right`,
- title block: `top-left` or `top-center`,
- footer: `bottom-left`, `bottom-center` or `bottom-right`.

An absent optional decoration occupies no band. Decoration spacing is deterministic and comes from the theme, not from View metadata.

Text measurement MUST use shipped or project-approved pinned fonts. If exact headless measurement cannot be made reproducible during Stage 0, the implementation MUST choose and document a deterministic metric strategy before Stage 1; host-dependent browser measurement is not acceptable for the release path.

Sanitized-logo hashes cover the canonical standalone asset. During embedding, the compositor MUST deterministically rename any asset IDs that collide with body or decoration IDs and update all fragment references. Collision handling MUST NOT mutate the stored standalone sanitized bytes or their manifest hash. Generated title and logo accessibility nodes use the same collision-safe ID allocator.

When title or logo metadata is present, the final SVG SHOULD expose deterministic `title`/`desc` and ARIA references supported by the selected consumer baseline. Visible title text and the theme asset `alt` value remain escaped plain text.

---

## 13. Theme and asset processing

### 13.1. Theme loading

A Theme/Brand Pack is loaded as one immutable resolved object:

- metadata and schema version,
- design tokens,
- semantic-to-token mappings,
- typography definitions,
- decoration tokens,
- declared brand assets,
- licenses.

Raw RGB or HEX values are permitted only in `theme.yaml`. Project configuration references an asset ID, not a file path. The resolver confirms that all referenced files are declared, remain below the global byte cap and have an accepted media type.

Per-asset `maxWidth` and `maxHeight` are layout bounds in CSS pixels. They are not security controls. Separate implementation-wide limits cap source bytes, intrinsic dimensions, element count, nesting depth, path complexity and filter regions.

### 13.2. SVG logo sanitizer

The sanitizer implements the normative `flowframe-svg-logo/v1` profile defined in `rules/svg-logo-profile-v1.md`. It is an allowlist validator and canonicalizer, not a repair tool.

Processing order:

1. hash original bytes,
2. enforce byte limit,
3. parse XML with DTD, entity expansion and network access disabled,
4. enforce element, depth, path-data and filter-complexity limits,
5. validate every element, namespaced attribute and presentation attribute,
6. parse `<style>` with a real CSS parser and validate selectors, properties and values,
7. reject scripts, events, animation, embedded HTML/media, raster images and all external references,
8. allow only internal fragment references and `url(#id)` values,
9. validate gradients, clipping, masks and the bounded filter subset,
10. reject the entire asset on any unsupported construct,
11. rewrite IDs and references deterministically,
12. canonicalize namespaces, numbers, attribute order and serialization,
13. hash canonical sanitized bytes.

Allowed and rejected constructs are defined normatively in the profile document, not duplicated in code comments. Unknown elements, attributes, CSS properties and filter primitives fail closed.

The manifest records profile ID, sanitizer implementation/version, original SHA-256 and sanitized SHA-256 for each used asset. Identical bytes, profile and implementation version MUST produce byte-identical sanitized output.

---

## 14. SVG normalization and verification

The final SVG verifier MUST ensure:

- one root `svg` element with a valid finite view box,
- no scripts, events, `foreignObject`, external URLs or undeclared namespaces,
- no remote font or image references,
- all fragment references resolve,
- title/logo/footer bounds remain inside the final canvas,
- the semantic body is not clipped by decoration bands,
- accessibility references resolve and visible metadata remains plain text,
- numeric values are finite and within configured canvas limits.

Normalization SHOULD make snapshots stable by fixing namespace prefixes, number precision, attribute order and ignorable metadata. It MUST NOT reorder visible child elements when painter order affects appearance.

Byte-identical final SVG is guaranteed only for the same FlowFrame version, sanitizer version, theme/assets, pinned fonts, D2 version, layout engine/version and renderer flags. The manifest makes that equivalence class explicit.

---

## 15. Manifest and provenance

Manifest data is accumulated during the build and written last. The collector MUST receive:

- FlowFrame version,
- logical relative source paths, schema versions and raw SHA-256 values,
- resolved configuration or built-in configuration ID/version,
- resolved theme identity, origin and aggregate hash,
- used asset processing records,
- renderer and layout versions/options,
- generated D2 and SVG hashes,
- requested output identity,
- generation timestamp.

Manifest paths are relative to the build root as required by the PRD; absolute host paths are forbidden. Map keys and arrays with set semantics are serialized in stable order. JSON uses UTF-8, fixed indentation and one trailing newline.

`generatedAt` is provenance, not an input to D2 or SVG. Reproducibility tests compare D2 and normalized SVG directly and compare manifests after excluding only fields explicitly declared volatile by the manifest schema.

---

## 16. CLI behavior

Initial commands:

```text
flowframe validate --model MODEL --view VIEW [--config CONFIG]
flowframe compile  --model MODEL --view VIEW [--config CONFIG] --output FILE
flowframe render   --input D2 --output SVG [--layout elk]
flowframe build    --model MODEL --view VIEW [--config CONFIG] --output-dir DIR
flowframe compare-layouts ...
flowframe review ...
flowframe version
```

`validate` runs all checks that do not require D2 rendering. `compile` validates and emits D2. `render` is a low-level controlled rendering command and does not accept arbitrary remote assets. `build` executes the complete transactional pipeline and is the normal user command.

Human diagnostics go to stderr. Machine-readable output MAY be selected with `--format json` and MUST preserve diagnostic codes and locations. Successful artifact paths MAY be printed to stdout. The low-level `render` command applies the same D2 policy lint as `compile` output, including rejection of remote resources and unsupported imports, before it invokes D2.

Exit codes follow the PRD contract:

| Code | Meaning |
|---:|---|
| 0 | Success |
| 1 | Validation or compilation error |
| 2 | Invalid command usage |
| 3 | Missing external dependency |
| 4 | Renderer failure or timeout |
| 5 | Policy or security violation |

An unexpected internal failure is reported with an `FFX` diagnostic and returns code 1 in v0.1; it MUST NOT be presented as a user-input validation error. Exact code-to-category assignments MUST be covered by CLI contract tests. A command returning non-zero MUST not claim that outputs are current.

---

## 17. Python API boundary

The CLI calls a small facade so tests and future integrations can use the same behavior:

```python
validate(request: ValidationRequest) -> ValidationResult
compile_view(request: CompileRequest) -> CompileResult
build(request: BuildRequest) -> BuildResult
```

Request and result types are immutable. Expected user errors are returned as diagnostics, not raised across the facade. Exceptions are reserved for programming faults and converted by the CLI into `FFX` diagnostics with sensitive paths removed unless debug mode is explicitly enabled.

No stability guarantee is made for lower-level Python modules in v0.1.

---

## 18. Security boundaries

All YAML, theme and SVG files are untrusted input. Required controls include:

- safe YAML loading, duplicate-key rejection and resource limits,
- schema allowlists with closed objects,
- project-root containment and symlink escape checks,
- no URL fetching,
- no shell interpolation,
- explicit D2 executable and argument list,
- process timeout and output limits,
- hardened XML parsing,
- fail-closed SVG/CSS/filter allowlists,
- transactional publication of artifacts,
- no secret values or absolute paths in normal diagnostics and manifests.

CI SHOULD run the renderer with a read-only source tree, a writable isolated build directory, no network and a constrained process identity.

---

## 19. Testing strategy

### 19.1. Fast tests

- schema positive/negative fixtures,
- loader resource-limit and source-location tests,
- semantic validation and deterministic diagnostic ordering,
- selector and projector unit tests,
- footer grammar tests,
- D2 escaping and writer golden tests,
- theme resolution and token-mapping tests,
- sanitizer allowlist, rejection and canonicalization tests,
- manifest serialization tests.

### 19.2. Integration tests

- complete architecture, flow and sequence builds with pinned D2,
- offline rendering,
- timeout and renderer-error behavior,
- project versus built-in theme resolution,
- transactional output behavior,
- deterministic repeated builds,
- global branding consistency across every family,
- decoration sizing and slot-conflict cases.

### 19.3. Snapshot and security tests

- normalized SVG snapshots for the golden corpus,
- bounds checks for long and Unicode labels,
- title/subtitle/footer/logo combinations,
- gradients, clips, masks and supported bounded filters,
- XXE, external URL, event handler, CSS import and filter-bomb rejection,
- path traversal and symlink escape attempts,
- fuzz/property tests for footer parsing, D2 escaping and SVG references.

Visual snapshots require intentional approval when the pinned renderer changes. A snapshot update alone is not evidence that a visual regression is acceptable.

---

## 20. Performance and observability

Stage 0 MUST record baseline timings and memory for small and medium golden diagrams. v0.1 performance budgets are then added to CI as generous regression limits, not hard real-time guarantees.

Debug logs MAY include stage timings, selected counts, renderer command metadata and cache decisions. Normal mode remains concise. Logs MUST never include unredacted environment variables or source contents by default.

Caching is optional for v0.1. If introduced, its key MUST include all source hashes, FlowFrame version, schema versions, resolved theme/asset hashes, sanitizer identity, D2/layout identity, flags and platform-relevant font identity. An incomplete cache key is worse than no cache.

---

## 21. Compatibility and versioning

Schema version changes follow these rules:

- additive optional fields MAY retain the same version if old consumers remain correct,
- changed meaning, removed fields or newly required fields require a new contract version,
- FlowFrame MUST reject unsupported major contract versions with a migration-oriented diagnostic,
- generated artifacts record both contract and tool versions,
- automatic migration is post-MVP; v0.1 may provide only guidance.

Theme packs declare their own version independently of FlowFrame. The sanitizer profile and sanitizer implementation version are both recorded because a profile-compatible bug fix can still affect canonical bytes.

---

## 22. Stage 0 decisions still required

Implementation MUST not guess these values:

1. exact pinned D2 version and installation/checksum strategy,
2. deterministic text-measurement approach and supported font formats,
3. SVG normalization library/algorithm after real D2 output is sampled,
4. numeric global limits for inputs, logos, XML complexity, canvas size, subprocess output and timeout,
5. initial generic icon set and license,
6. supported documentation renderers/browsers,
7. committed-versus-CI-generated SVG policy,
8. release performance budgets,
9. minimum AI evaluation threshold.

Each decision is captured as an ADR with context, tested alternatives, decision and consequences. The actionable order and acceptance gates are defined in [`implementation-plan.md`](implementation-plan.md).
