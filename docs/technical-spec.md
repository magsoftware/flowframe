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
- produce byte-identical D2 for identical inputs and pinned toolchain,
- produce reproducible SVG when the complete pinned rendering toolchain is identical,
- work without network access after installation in the deterministic core (optional AI adapters may use a network),
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

Dependency versions MUST be locked for releases. D2 itself MUST be installed at the version selected by the Stage 0 ADR and verified by checksum. Resolve `FLOWFRAME_D2` (explicit absolute executable path) before PATH and verify the chosen executable's version and SHA-256 against the packaged platform-specific pin on every rendering invocation. Only official release executables matching the packaged per-platform pin are supported; distribution rebuilds with different bytes are not. A pinned installer helper MUST obtain and verify the approved release artifact explicitly, with an offline installation path. Builds MUST NOT download D2 automatically. Missing executable, version mismatch and checksum mismatch have distinct dependency diagnostics, all returning 3 with installation guidance. Updating the accepted pin requires a FlowFrame release. This applies to all builds, without a special reproducible/CI mode.

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
├── review/
│   └── deterministic.py
├── adapters/
│   └── ai/                 # optional P7 dependency group
├── manifest/
│   ├── collector.py
│   └── writer.py
└── resources/
    ├── schemas/
    ├── themes/flowframe-light/
    ├── fonts/flowframe-default/
    ├── mappings/v1.json
    ├── d2/                 # internal templates; never output imports
    └── icons/generic/
```

Public entry points SHOULD be limited to the CLI and a small facade in `api.py`. Internal modules MAY change before v1 without compatibility guarantees.

---

## 6. Public contracts and resolution

### 6.1. Contract files

The canonical schemas are:

- `src/flowframe/resources/schemas/flowframe-config.schema.json` for `flowframe-config/v1`,
- `src/flowframe/resources/schemas/system-model.schema.json` for `flowframe-model/v1` System Models,
- `src/flowframe/resources/schemas/view.schema.json` for `flowframe-view/v1` View Specifications,
- `src/flowframe/resources/schemas/flowframe-theme.schema.json` for `flowframe-theme/v1`,
- `src/flowframe/resources/schemas/flowframe-manifest.schema.json` for `flowframe-manifest/v1`.

Schemas MUST set `additionalProperties: false` at every closed object boundary. Extensions, if later supported, require a dedicated namespaced field rather than accepting misspelled properties.

### 6.2. Project root

Resolution is deterministic:

1. If `--config` is given, resolve that file and use its parent as project root.
2. Otherwise walk upward from the System Model directory, checking through the nearest VCS root (`.git` file/directory), for the nearest `flowframe.yaml`. Outside VCS check only the model directory; an ancestor config requires explicit `--config`.
3. If none exists, use the System Model directory as the effective root and create the versioned built-in configuration in memory.
4. Resolve all project-relative paths against that root, never against the process working directory.

With no configuration the model directory is the project root. An out-of-root View diagnostic MUST suggest placing `flowframe.yaml` in the common ancestor of model and View and, if necessary, passing `--config`; sibling `model/` and `views/` directories are supported with that explicit common root.

The resolver MUST canonicalize paths, reject traversal outside the project root for project-owned assets and retain both a logical relative path and a resolved filesystem path. Symlinks that escape the project root MUST be rejected for themes and assets.

### 6.3. Theme resolution

For theme ID `X`:

1. if X is a built-in ID, reject a conflicting project directory and resolve immutable `resources/themes/X/theme.yaml`,
2. otherwise resolve `<project-root>/themes/X/theme.yaml`,
3. require the declared theme ID to equal X and fail if no such theme exists.

File-level overlay or merging between a project theme and a built-in theme is forbidden in v0.1. Missing element, boundary or relation mappings use `resources/mappings/v1.json`, containing the complete PRD §13.3 mapping table independently of any theme. The resulting token reference MUST exist in the selected theme. Missing files, malformed tokens or unknown token references never trigger a switch to another theme.

The manifest MUST record theme ID, declared version, origin (`project` or `built-in`) and an aggregate hash. The aggregate is calculated from a domain tag followed by each normalized POSIX relative path, byte length and raw file bytes for `theme.yaml`, the required license inventory and all referenced assets, sorted by path. Built-in font-set references contribute package resource IDs and raw bytes. Length prefixes prevent ambiguous concatenation.

---

## 7. Loading and source locations

The loader pipeline is:

1. read bytes with an explicit UTF-8 policy,
2. accept and remove one leading UTF-8 BOM for parsing; raw provenance hashes still include it,
3. enforce input byte and YAML nesting limits,
4. compose a YAML node graph without invoking custom constructors,
5. allow only the documented core scalar, sequence and mapping tags,
6. reject duplicate keys, aliases exceeding the configured expansion limit and non-string mapping keys where the schema expects objects,
7. construct JSON-compatible primitive values in FlowFrame code; normalize schema-designated human-text values to NFC before length validation, but do not normalize IDs, paths, keys or raw source hashes,
8. build a JSON Pointer to source-location map,
9. validate against the selected schema,
10. construct immutable domain objects.

The location map stores file, one-based line and column, plus the nearest JSON Pointer. Semantic validation errors MUST refer to the most specific source field available.

Input order MAY be retained for author-friendly output, but semantic equality and generated ordering MUST not depend on YAML parser implementation details.

---

## 8. Validation model

Validation runs in layers and stops only where continuing would create misleading diagnostics:

1. safe YAML loading and project/configuration schema,
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

IDs use the PRD §8.3 grammar and one namespace for the system, boundaries, elements, relations, scenarios and all steps. Duplicate detection occurs before references are resolved. Containment cycles MUST be detected with a deterministic depth-first traversal and reported as the shortest reproducible cycle path available.

For an undirected relation, `source` and `target` remain mandatory. Directionality affects arrowheads and eligible labels only; it does not change the storage model.

Relation endpoints and all scenario participants reference elements, never boundaries. Scenario `relationId`, protocol inheritance and participant permutations follow PRD §§9.4 and 10.5. Parents reference only boundaries/system; omitted parent means outside the system, without kind-based inference. Validate actor/external-system ancestry. When systemBoundary is enabled but no selected element reaches the system through its parents, emit the PRD FFV warning and omit that empty boundary.

View validation covers family/subtype, scenario existence, selection element/relation IDs, forbidden family fields, traversal bounds, empty results and layout compatibility. Relation combinations and display defaults are exactly PRD §§9.3.1 and 10.8, not inferred independently by projectors.

### 8.2. Diagnostics

A diagnostic contains:

```text
severity: error | warning | info
code: stable FlowFrame code
message: concise human explanation
category: validation | usage | dependency | execution | policy | internal
location: optional file + one-based line + column + JSON Pointer
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

Source-related diagnostics MUST include their closest valid location; renderer/output/internal failures may omit it. CLI text uses `system-model.yaml:42:9 /relations/2/target`. `docs/diagnostics.md` is the implementation-time code registry and records each code's severity/category and example.

Codes and exit statuses are public CLI behavior. Message wording MAY improve without a major schema change.

---

## 9. Selection and projection

### 9.1. Selection

Selection MUST be a pure function of validated model and View. The authoritative algorithm is PRD §10.7: AND between populated element filters, OR within each; exclusions remove seeds and block traversal; relation filters constrain both traversal and final edges; breadth-first expansion is bidirectional with depth 1–10. Empty lists are unpopulated and no populated element criterion seeds all elements. Empty final selections fail with FFV; explicit include/exclude conflicts and disconnected results warn.

The selector returns selected elements, ancestors, candidate relations, inclusion reasons and diagnostics in stable source order with ID tie-breaks. No hash-map order may leak into output. Only ancestors of retained nodes are added, so no separate empty-boundary pruning pass is necessary.

Sequence uses only its scenario and an optional exact participant permutation (PRD §10.5); selection/exclusion and boundaries are forbidden. Resolve first-occurrence ordering, including note-only participants and from-before-to order, before building Sequence IR.

Configuration defaults, family display fields and integration-flow defaults come from PRD §§10.8 and 13.2; implementations MUST share one resolver rather than repeat defaults in CLI/projectors.

### 9.2. Family projectors

Each projector maps the selection to one of three disjoint IRs:

- Architecture IR: nodes, nested boundary groups, typed edges and portable layout hints.
- Flow IR: selected nodes, directed/bidirectional semantic flows, labels, explicit payload and protocol metadata. A selected undirected dependency is an FFV error, not a warning or omitted edge. Store annotations derive from kind; source/sink annotations derive only from selected degree, never from guessed business roles.
- Sequence IR: ordered participants, messages (including self-messages) and notes; groups and activation spans are post-MVP. Messages use a solid line and one from-to arrowhead without timing semantics; responses are separate reverse messages and relationId does not import relation appearance. Optional legends explain only message/note/numbering constructs.

Deprecated elements retain their `[deprecated]` text marker in labels and accessible descriptions in every family. Architecture/flow `display.relationLabels` hides only user labels; mandatory semantic labels remain visible.

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

- write a generated-file header with FlowFrame version and resolved theme fingerprint equal to the manifest's `theme.sha256`, computed by the same aggregate algorithm (not a D2 content hash or authenticity token),
- emit only constructs owned by the relevant family generator,
- escape IDs, labels and string values in one audited module,
- serialize properties in a fixed order,
- serialize nodes before edges,
- preserve IR order and use stable ID tie-breakers,
- resolve all visual values through theme tokens and framework mappings,
- avoid timestamps, absolute paths and process-specific values,
- finish files with exactly one newline.

Raw D2 snippets from YAML are forbidden. Unknown tokens, kinds or semantics are validation errors rather than pass-through content.

Generated identifiers MUST prefix source IDs in a reserved internal namespace and escape all D2 syntax. D2 keywords remain valid source IDs; source-to-generated mapping is injective. Derived IDs use separate namespaces and stable semantic identity, never random UUIDs or Python hashes.

Generated D2 is self-contained: classes are materialized from resolved theme tokens/mappings and internal templates, with approved generic icons embedded as sanitized bounded SVG data URIs. There are no output imports, remote URLs or filesystem asset paths. The v0.1 generic index maps kinds, not free-text technologies, to assets with hashes/licenses. Deduplicate embedded URI declarations by sanitized asset SHA-256, using reusable classes/declarations so each unique asset appears once in D2 regardless of node count. An explicit null entry is a silent semantic-shape fallback; missing entries or missing declared files are invalid-pack errors. P0 verifies reuse with the pinned D2, including interactions with per-kind styling.

Required D2 lint rules reject missing/invalid generated headers, imports, absolute or relative filesystem references, remote URLs, raw source-injected constructs, markdown/HTML labels, and unapproved data URIs. Only compiler-produced bounded SVG icon URIs with validated vector content are allowed. The lint validates content, not just a trusted-looking header.

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

The wrapper MUST record the verified executable SHA-256, reported D2 version, layout engine and version if exposed, arguments and normalized failure details. A timeout or non-zero exit is an error even if a partial SVG exists. Partial outputs MUST be removed or left only in a diagnostic temporary directory, never published as successful artifacts.

`d2 validate` is the intended syntax boundary for render/build. Stage 0 MUST verify the pinned release's command and exits and retain local version-specific observations in its ADR. `d2 fmt --check` is not a substitute. Compile does not invoke D2. A syntax rejection for freshly generated build input is FFX/internal (6); rejection of a separately supplied render artifact is FFD/validation (1). Classify a crash/timeout as execution (4), not a syntax rejection, and retain bounded process evidence for diagnostics.

### 11.1. Artifact publication

Individual compile/render outputs use a sibling temporary file and `os.replace` after successful verification. Complete builds publish to an ordinary dedicated output directory, without symlinks or generation history.

For target `<parent>/<name>`, private control state lives in `<parent>/.<name>.flowframe/`: one stable advisory-lock file, one staging directory, at most one previous-output directory and a small recovery journal. The installer/docs mark this control directory as untracked build state; it is not part of the three public artifacts. Staging and target MUST be on the same filesystem. Existing control state must have valid ownership records for this exact target; a name alone never authorizes recursive cleanup.

1. Canonicalize the parent and target identity, reject an output symlink and acquire the per-target OS advisory lock. A competing writer fails immediately with a stable FFR execution diagnostic (4). Do not delete the lock file to break a held lock.
2. Recover an interrupted prior publication before starting work. If the target is missing and the recorded intact backup exists, restore it. If the target contains the verified newly published set, recognize completion. Do not replace unrelated content encountered at the target or control paths; report the conflict and preserve the backup.
3. Accept an absent target, an empty ordinary directory or a previous FlowFrame output whose manifest/schema, exact file inventory and output hashes validate. Reject unrelated files or modified artifacts with guidance to choose a dedicated directory. Never claim ownership from filenames alone.
4. Prepare D2/SVG in the single staging directory, write the manifest last, verify the complete artifact set and flush it before publication. Failure here leaves the old target unchanged. Remove only staging state owned by this run.
5. Before replacement, clean the older retained backup if it is verified as FlowFrame-owned; on cleanup failure stop before touching the current target. Persist recovery intent, rename the old target to the backup slot (if one exists), then rename staging to the target. Record completion. First publication needs only the second rename; an accepted empty target can occupy the temporary backup slot.
6. On failure/SIGINT during replacement, attempt to restore the recorded backup when no new target was published. Report an unsuccessful rollback and the backup/recovery location explicitly. A crash is resolved by the same journal-aware recovery on the next invocation.
7. Retain at most the current output, one previous output and one in-progress staging set. Automatically remove obsolete owned state under the lock; no unbounded accumulation or cleanup command is required. An empty backup can be removed after success.

Two renames are not an atomic directory swap: readers may briefly see no target, and separate concurrent file opens have no snapshot guarantee. Consumers should read/copy/commit ordinary output files after a successful build, rather than reading through a publication in progress. Validation/render failure before publication preserves the old output; rollback after a publication failure is best effort, with explicit recovery status, not an unconditional durability promise. Output I/O, lock conflicts and failed recovery use execution category/code 4; unsafe path redirection remains policy/code 5.

P0 tests rename failure, crash windows and ownership checks on supported filesystems. P3 implements this bounded state machine; no optional symlink mode, content-addressed generations, exported copies or whole-project transaction is in scope.

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
3. wrap all decoration text according to the PRD, then render title and subtitle as one title block using pinned theme fonts,
4. sanitize, canonicalize and measure the selected logo; require a finite intrinsic view box,
5. expand the logo proportionally into its per-asset `maxWidth` × `maxHeight` display box,
6. render the footer fragment,
7. compute the top band as the maximum occupied top-slot height plus `decoration-gap` above and below,
8. compute separate stacked legend/footer bands, each with `decoration-gap` above and below,
9. calculate slot-aware canvas width using the exact PRD §13.6 formula (including symmetric clearance for a centered title),
10. translate the semantic body without changing its internal geometry,
11. place decorations into their configured slots and embed the sanitized logo nodes,
12. normalize and verify the complete SVG.

The title and subtitle share one primary top slot. A logo and title configured for the same top slot are invalid. The initial allowed slots are:

- logo: `top-left` or `top-right`,
- title block: `top-left` or `top-center`,
- footer: `bottom-left`, `bottom-center` or `bottom-right`.

An absent optional decoration occupies no band. Decoration spacing is deterministic and comes from the theme, not from View metadata.

Text measurement MUST use shipped or project-approved pinned fonts. If exact headless measurement cannot be made reproducible during Stage 0, the implementation MUST choose and document a deterministic metric strategy before Stage 1; host-dependent browser measurement is not acceptable for the release path.

Deterministic measurement does not by itself guarantee identical consumer rendering. The Stage 0 ADR MUST select one output-wide text policy:

- preserve accessible/searchable SVG `<text>` and pin or embed fonts, accepting documented rasterization differences between consumers, or
- convert visible text to deterministic paths, accepting larger output and loss of native selection/search while providing per-object accessible names/descriptions plus a generated textual summary.

The policy applies to D2 body text and FlowFrame decorations. Path conversion cannot be selected unless the accessibility baseline remains satisfied. A hybrid policy is allowed only if the ADR defines a clear boundary and tests both portability and accessibility; it MUST NOT arise accidentally from two independent render paths.

Sanitized-logo hashes cover the canonical standalone asset. During embedding, the compositor MUST deterministically rename any asset IDs that collide with body or decoration IDs and update all fragment references. Collision handling MUST NOT mutate the stored standalone sanitized bytes or their manifest hash. Generated title and logo accessibility nodes use the same collision-safe ID allocator.

When title or logo metadata is present, the final SVG MUST expose deterministic `title`/`desc` and ARIA references supported by the selected consumer baseline. Visible title text and the theme asset `alt` value remain escaped plain text. Always emit a top-level `<desc>` with deterministic selected-element/relation or ordered-scenario summary and per-object accessible descriptions. A missing visible title uses the View ID as its accessible name. Legends describe used non-color relation appearances, use pinned theme text, and occupy a measured bottom band above the footer. Their space is included before final sizing.

PRD §13.3 defines numeric token units, required colors, default mappings and typography. P0 verifies the required custom TTF faces with D2; fonts used by body and compositor are pinned consistently. Title/subtitle/footer text is NFC-normalized, measured and greedily wrapped at whitespace, with grapheme-boundary fallback, under `decoration-text-max-width`. Body bounds include the pinned D2 `--pad`; do not crop that padding again. Blocks in a top band are vertically centered, and horizontal edges respect `decoration-padding-x`.

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

The source theme requires color tokens but may omit numeric decoration tokens; fill their PRD defaults to create the resolved theme. Omitted typography resolves to the built-in font set. Explicit typography is a closed exclusive union of `fontSet` and `fonts`, with custom faces nested under `typography.fonts`; neither an empty object nor both variants are valid.

Raw RGB or HEX values are permitted only in `theme.yaml`. Project configuration references an asset ID, not a file path. The resolver confirms that all referenced files are declared, remain below the global byte cap and have an accepted media type.

Per-asset `maxWidth` and `maxHeight` are layout bounds in CSS pixels. They are not security controls. Separate implementation-wide limits cap source bytes, intrinsic dimensions, element count, nesting depth, path complexity and filter regions.

### 13.2. SVG logo sanitizer

The sanitizer implements the normative `flowframe-svg-logo/v1` profile defined in `rules/svg-logo-profile-v1.md`. It is an allowlist validator and canonicalizer, not a repair tool.

Processing order:

1. hash original bytes,
2. enforce byte limit,
3. parse XML with DTD, entity expansion and network access disabled, enforce bounded whole-document active-content/URL checks and remove only PRD-allowlisted inert comments/editor metadata,
4. enforce element, depth, path-data and filter-complexity limits,
5. validate every element, namespaced attribute and presentation attribute,
6. parse `<style>` with a real CSS parser and validate selectors, properties and values,
7. reject scripts, events, animation, embedded HTML/media, raster images and all external references,
8. allow only internal fragment references (including `xlink:href`, with conflicting href forms rejected) and `url(#id)` values,
9. validate gradients, clipping, masks and the bounded filter subset,
10. reject the entire asset on any unsupported construct,
11. rewrite IDs and references deterministically,
12. canonicalize namespaces, lossless numeric values, attribute order and serialization without rounding geometry,
13. hash canonical sanitized bytes.

Allowed and rejected constructs are defined normatively in the profile document, not duplicated in code comments. Unknown rendering elements, attributes, CSS properties and filter primitives fail closed; only the explicit inert-metadata exception in PRD §16.5 permits removal.

The manifest records profile ID, sanitizer implementation/version, original SHA-256 and sanitized SHA-256 for each used asset. Identical bytes, profile and implementation version MUST produce byte-identical sanitized output.

---

## 14. SVG normalization and verification

The final SVG verifier MUST ensure:

- one root `svg` element with a valid finite view box,
- no scripts, events, `foreignObject`, external URLs or undeclared namespaces,
- no remote font or image references; permit only validated bounded embedded icon SVGs and renderer-emitted font data with expected MIME types and a documented source-font-to-output transformation,
- all fragment references resolve,
- title/logo/footer bounds remain inside the final canvas,
- the semantic body is not clipped by decoration bands,
- accessibility references resolve and visible metadata remains plain text,
- numeric values are finite and within configured canvas limits.

Normalization SHOULD make snapshots stable by fixing namespace prefixes, lossless numeric serialization, attribute order and explicitly ignorable metadata. It MUST NOT reorder visible child elements when painter order affects appearance. D2 may subset or re-encode fonts: source TTF hashes and embedded-font hashes need not be equal. P0 MUST characterize that pinned transformation; verification checks allowed MIME/content, bounds and provenance, and records both source and embedded hashes rather than demanding byte identity with the source TTF.

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
- optional `generatedAt` only when `SOURCE_DATE_EPOCH` supplies a valid nonnegative epoch, serialized as UTC RFC 3339 with `Z`.

Source paths are relative to the project root; output paths are relative to the manifest directory. Built-in resources use package resource IDs; absolute host paths are forbidden. Renderer flags record logical options/resource IDs instead of resolved absolute font paths. Every used icon/font records kind, ID, project path or package ID/version, license reference and SHA-256 in a `resources` array. `assetProcessing` is present exactly when vector logos/icons are sanitized by FlowFrame; font transformations are recorded with source/embedded hashes in `resources`, not as logo-sanitizer runs. Map keys and arrays with set semantics are serialized in stable order. JSON uses UTF-8, fixed indentation and one trailing newline.

No wall-clock timestamp is generated by default. With identical `SOURCE_DATE_EPOCH` and inputs the entire manifest is stable; no invented JSON Schema `volatile` keyword is needed. Reproducibility tests compare all fields under the same environment. Input hashes cover raw bytes even when parsing normalizes BOM/NFC.

---

## 16. CLI behavior

The exact signatures, defaults, category-to-exit mapping and precedence are defined once in [PRD §17](prd.md#17-cli-contract); CLI help and contract tests MUST follow that contract.

- `validate` supports model-only, model+view and exclusive theme-only forms; it never invokes D2.
- `compile` validates and emits self-contained D2 only. It requires no D2 executable.
- `render` verifies the generated header, theme fingerprint and full D2 lint, then runs D2 validation/rendering. It emits body SVG only. An explicit config is used for custom fonts/theme; without it only built-in configuration is used.
- `build` composes the complete branded diagram, adds accessible summary/legend and publishes one complete artifact set using recoverable ordinary-directory replacement. `--layout` overrides `render.layoutEngine`; only ELK is accepted in v0.1 and no fallback exists.
- `build --all` uses the explicit project `views` registry and a single supplied model, validates IDs/paths before rendering, sorts by View ID and publishes independently to `DIR/<view-id>/` using section 11.1 (no concurrent-reader atomicity). It stops on failure and reports already published views.
- `review` selects stages of the same validation service as `validate`, without duplicate rules or D2/AI. Default modes produce the same findings/status as model+view validate. P7 source-conformance uses explicitly supplied sources and an optional adapter; visual review is post-MVP.
- `compare-layouts` remains post-MVP, compiling once and rendering with at least two explicitly selected engines without fallback.

All processing commands support `--format text|json` and `--debug`. JSON diagnostics are one array on stderr; optional redacted debug objects are contained within diagnostic entries, never emitted as extra text. No implicit warning promotion is implemented.

Exit codes are 0 success, 1 validation, 2 usage, 3 dependency, 4 execution (renderer/timeout/output I/O/lock/publication), 5 policy, 6 internal and 130 SIGINT. Each diagnostic has a category independent of its prefix; observed-error precedence is 6 > 5 > 4 > 3 > 2 > 1. Cancellation returns 130 after cleanup. Unknown source properties are ordinarily validation errors, but known forbidden styling fields and unsafe resources are policy errors. Unexpected exceptions use FFX and never masquerade as invalid user input.

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
- staged, locked artifact publication with bounded backup and explicit recovery,
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
- recoverable output replacement, bounded retention and crash/lock/cleanup behavior,
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

- closed contracts require a new `schemaVersion` for any newly accepted field or enum value, including optional additions; a version covers the accepted language, not merely required fields,
- changed meaning, removed fields or newly required fields require a new contract version,
- FlowFrame MUST reject unsupported contract versions with a migration-oriented diagnostic,
- generated artifacts record both contract and tool versions,
- automatic migration is post-MVP; v0.1 may provide only guidance.

Theme packs declare their own version independently of FlowFrame. The sanitizer profile and sanitizer implementation version are both recorded because a profile-compatible bug fix can still affect canonical bytes.

---

## 22. Stage 0 decisions still required

Implementation MUST not guess these values:

1. exact pinned D2 version and installation/checksum strategy,
2. output-wide SVG text policy: deterministic measurement, `<text>` versus paths, font embedding/formats and accessibility consequences,
3. SVG normalization library/algorithm after real D2 output is sampled,
4. numeric global limits for inputs, logos, XML complexity, canvas size, subprocess output and timeout,
5. initial generic icon set and license,
6. supported documentation renderers/browsers,
7. committed-versus-CI-generated SVG policy,
8. release performance budgets,
9. offline icon embedding and D2 output font verification,
10. verification of PRD display defaults and portable profile mappings on representative output.

Each decision is captured as an ADR with context, tested alternatives, decision and consequences. The actionable order and acceptance gates are defined in [`implementation-plan.md`](implementation-plan.md).

The first AI adapter platform is chosen at P7.1; its representative corpus and threshold are frozen at P7.4 before release evaluation. `uv`/Hatchling, BOM acceptance and the PRD display defaults are already decided, not Stage 0 open choices.

Dark theme is deliberately absent from this list because it is post-MVP, consistent with the PRD and P9.
