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
| Language | Python 3.12+ | Type annotations are mandatory in production code. CI runs 3.12 and 3.13. |
| Packaging | `pyproject.toml` with `uv` | Lock application and development dependencies. Build wheels with Hatchling. Static package version (no VCS-derived versions). |
| CLI | Typer | Thin command layer; business logic stays callable without a terminal. |
| YAML | `ruamel.yaml` parser/composer | Retain node marks, allow only core YAML tags and construct JSON-compatible values in FlowFrame code. |
| Schema validation | `jsonschema`, Draft 2020-12 | JSON Schema files are the public structural contracts. |
| Internal models | frozen dataclasses and enums | Constructed only after structural validation; avoid a second public validation contract. |
| XML/SVG | hardened `lxml` parser | Disable entity resolution, DTD loading and network access. |
| CSS parsing | `tinycss2` | Required for allowlist validation of SVG `<style>` content. |
| Font metrics | selected by ADR-0007 (candidates `fontTools`, `uharfbuzz`) | Deterministic text measurement for labels and decorations; text-to-path conversion if selected. |
| Grapheme segmentation | `regex` (`\X`) | Wrapping fallback at grapheme boundaries. |
| Property and fuzz tests | `hypothesis` | Footer grammar, D2 escaping, ID mapping, selection invariants, SVG references. |
| SBOM | CycloneDX JSON generator | Release provenance inventory (ADR-0024). |
| Hashing | Python `hashlib.sha256` | Hash raw source bytes and canonical processed bytes as defined below. |
| D2 integration | pinned D2 CLI through `subprocess` | No shell invocation. Direct Go integration is deferred. |
| Tests | `pytest` | Unit, contract, integration, golden, snapshot and security suites. |
| Static quality | Ruff and mypy | Formatting/linting and strict checks for the core packages. |

Dependency versions MUST be locked for releases. D2 itself MUST be installed at the version recorded in `resources/d2-pin.json` ([ADR-0006](adrs/0006-pinned-d2-distribution.md)) and verified by checksum. Resolve `FLOWFRAME_D2` (explicit absolute executable path) before PATH and verify the chosen executable's version and SHA-256 against the packaged platform-specific pin on every rendering invocation. Only official release executables matching the packaged per-platform pin are supported; distribution rebuilds with different bytes are not. The installer helper `python -m flowframe.install_d2` obtains and verifies the approved release artifact explicitly, with an offline `--from-archive` path; it is delivered in P1.8 and used by CI. Builds MUST NOT download D2 automatically. Missing executable, version mismatch and checksum mismatch have distinct dependency diagnostics, all returning 3 with installation guidance. Updating the accepted pin requires a FlowFrame release. This applies to all builds, without a special reproducible/CI mode.

### 3.1. Supported platforms

v0.1 targets macOS and Linux on architectures for which the pinned D2 release is available, on local POSIX filesystems (ADR-0010). Advisory locking uses POSIX `fcntl`. Windows MAY work through Python and D2 but is not a release gate until a Windows CI job is explicitly added.

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
├── install_d2.py           # `python -m flowframe.install_d2` (ADR-0006)
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
│   ├── defaults.py         # single default resolver (ADR-0019)
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
│   ├── labels.py           # label composition (ADR-0011)
│   ├── line_styles.py      # shared with the legend (ADR-0015)
│   └── families/
├── rendering/
│   ├── d2_process.py
│   ├── svg_verify.py
│   └── normalization.py
├── presentation/
│   ├── theme_resolver.py
│   ├── footer_template.py
│   ├── text_fragments.py
│   ├── text_metrics.py     # shared measurement and wrapping (ADR-0007)
│   ├── contrast.py         # single contrast validator (ADR-0012)
│   ├── legend.py
│   ├── accessibility.py    # title/desc/ARIA injection (ADR-0014)
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
    ├── defaults/v1.json    # ADR-0019
    ├── limits/v1.json      # ADR-0009
    ├── d2-pin.json         # ADR-0006
    ├── d2/                 # internal templates; never output imports
    └── icons/generic/      # pre-sanitized at package build (ADR-0013)
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
- `src/flowframe/resources/schemas/flowframe-manifest.schema.json` for `flowframe-manifest/v1`,
- `src/flowframe/resources/schemas/flowframe-diagnostics.schema.json` for `flowframe-diagnostics/v1` JSON diagnostics (ADR-0016),
- `src/flowframe/resources/schemas/flowframe-d2-pin.schema.json` for `flowframe-d2-pin/v1` (ADR-0006).

Schemas MUST set `additionalProperties: false` at every closed object boundary. Extensions, if later supported, require a dedicated namespaced field rather than accepting misspelled properties. Versioning and multi-version support follow [ADR-0004](adrs/0004-schema-authority-and-contract-versioning.md). JSON Schema `default` annotations are informative; defaults are applied only by `domain/defaults.py` from `resources/defaults/v1.json` ([ADR-0019](adrs/0019-default-value-resolution.md)).

### 6.2. Project root

Resolution is deterministic:

1. If `--config` is given, resolve that file and use its parent as project root.
2. Otherwise walk upward from the System Model directory, checking through the nearest VCS root (`.git` file/directory), for the nearest `flowframe.yaml`. Outside VCS check only the model directory; an ancestor config requires explicit `--config`.
3. If none exists, use the System Model directory as the effective root and create the versioned built-in configuration in memory.
4. Resolve all project-relative paths against that root, never against the process working directory.

With no configuration the model directory is the project root. An out-of-root View is an FFC validation error (1); a symlink or asset escaping the root is a policy error (5). The out-of-root View diagnostic MUST suggest placing `flowframe.yaml` in the common ancestor of model and View and, if necessary, passing `--config`; sibling `model/` and `views/` directories are supported with that explicit common root.

The resolver MUST canonicalize paths, reject traversal outside the project root for project-owned assets and retain both a logical relative path and a resolved filesystem path. Symlinks that escape the project root MUST be rejected for themes and assets.

### 6.3. Theme resolution

For theme ID `X`:

0. reject the project if any directory `<project-root>/themes/<built-in-id>/` exists, whichever theme is selected,
1. if X is a built-in ID, resolve immutable `resources/themes/X/theme.yaml`,
2. otherwise resolve `<project-root>/themes/X/theme.yaml`,
3. require the declared theme ID to equal X and fail if no such theme exists.

File-level overlay or merging between a project theme and a built-in theme is forbidden in v0.1. Missing element, boundary or relation mappings use `resources/mappings/v1.json`, containing the complete PRD §13.3 mapping table independently of any theme. The resulting token reference MUST exist in the selected theme. Missing files, malformed tokens or unknown token references never trigger a switch to another theme.

The manifest MUST record theme ID, declared version, origin (`project` or `built-in`) and an aggregate hash. The aggregate covers `theme.yaml`, the license inventory, every declared asset and every custom font file; built-in font-set files contribute resource names and raw bytes. The exact encoding (domain tag `flowframe-theme-hash/v1\n`, 4-byte big-endian name length, name, 8-byte big-endian content length, content, entries sorted bytewise by name) and a committed test vector are defined in [ADR-0020](adrs/0020-theme-aggregate-hash.md).

---

## 7. Loading and source locations

The loader pipeline is:

1. read bytes with an explicit UTF-8 policy,
2. accept and remove one leading UTF-8 BOM for parsing; raw provenance hashes still include it,
3. enforce the fixed input byte and YAML nesting limits (ADR-0009),
4. compose a YAML 1.2 node graph without invoking custom constructors; reject a `%YAML 1.1` directive (ADR-0018),
5. allow only the YAML 1.2 core-schema scalar, sequence and mapping tags; reject merge keys (`<<`) and custom tags,
6. reject duplicate keys, aliases exceeding the fixed expansion limit and non-string mapping keys where the schema expects objects,
7. construct JSON-compatible primitive values in FlowFrame code; normalize schema-designated human-text values to NFC before length validation (PRD §8.4 limits and control-character rule), but do not normalize IDs, paths, keys or raw source hashes,
8. build a JSON Pointer to source-location map,
9. validate against the selected schema,
10. construct immutable domain objects.

The location map stores file, one-based line and column, plus the nearest JSON Pointer. Semantic validation errors MUST refer to the most specific source field available.

Input order MAY be retained for author-friendly output, but semantic equality and generated ordering MUST not depend on YAML parser implementation details.

---

## 8. Validation model

Validation runs in layers and stops only where continuing would create misleading diagnostics. Which layers each command runs is defined by the command stage matrix in [ADR-0017](adrs/0017-command-stage-matrix.md); `validate`, `review` and `build` share one validation service and MUST report identical diagnostics up to and including layer 8.

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
12. decoration, legend and accessibility composition,
13. output SVG canonicalization, safety and integrity checks,
14. manifest schema validation.

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

Relation endpoints and all scenario participants reference elements, never boundaries. Scenario `relationId`, protocol inheritance and participant permutations follow PRD §§9.4 and 10.5. Parents reference only boundaries/system; omitted parent means outside the system, without kind-based inference. Validate actor/external-system ancestry and require `system.id` in the ancestor chain of every `subsystem` boundary. When systemBoundary is enabled but no selected element reaches the system through its parents, emit the PRD FFV warning and omit that empty boundary.

View validation covers family/subtype, scenario existence, selection element/relation IDs, forbidden family fields, traversal bounds, empty results (by running the selector of §9.1), `subtitle` without `title` and layout compatibility. Relation combinations and display defaults are exactly PRD §§9.3.1 and 10.8, not inferred independently by projectors.

### 8.2. Diagnostics

A diagnostic contains (JSON form: `flowframe-diagnostics/v1`, [ADR-0016](adrs/0016-diagnostics-contract.md)):

```text
severity: error | warning | info
code: stable FlowFrame code
message: concise human explanation
category: validation | usage | dependency | execution | policy | internal
location: optional file (project-root-relative) + one-based line + column + JSON Pointer
related: zero or more related locations
hint: optional corrective action
debug: optional object, only with --debug
```

Diagnostics are sorted by file, line, column, code and message; diagnostics without a location follow, sorted by code and message.

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
| `FFR` | Renderer, SVG output and publication |
| `FFA` | AI adapter and source-conformance findings |
| `FFX` | Internal/unexpected failures |

Source-related diagnostics MUST include their closest valid location; renderer/output/internal failures may omit it. CLI text uses `system-model.yaml:42:9 /relations/2/target`. `docs/diagnostics.md` is the implementation-time code registry and records each code's severity/category and example; codes used only in prereleases are marked `prerelease-only` and removed before v0.1.

Codes and exit statuses are public CLI behavior. Message wording MAY improve without a major schema change.

---

## 9. Selection and projection

### 9.1. Selection

Selection MUST be a pure function of validated model and View. The authoritative algorithm is PRD §10.7: AND between populated element filters, OR within each; exclusions remove seeds and block traversal; relation filters constrain both traversal and final edges; breadth-first expansion is bidirectional with depth 1–10. Empty lists are unpopulated and no populated element criterion seeds all elements. Empty final selections fail with FFV; explicit include/exclude conflicts and disconnected results warn.

The selector returns selected elements, ancestors, candidate relations, inclusion reasons and diagnostics in stable source order with ID tie-breaks. No hash-map order may leak into output. Only ancestors of retained nodes are added, so no separate empty-boundary pruning pass is necessary.

Sequence uses only its scenario and an optional exact participant permutation (PRD §10.5); selection/exclusion and boundaries are forbidden. Resolve first-occurrence ordering, including note-only participants and from-before-to order, before building Sequence IR.

Configuration defaults, family display fields and integration-flow defaults come from PRD §§10.8 and 13.2 and are stored once in `resources/defaults/v1.json`; `domain/defaults.py` is the only resolver (ADR-0019). CLI, projectors and generators never repeat defaults.

The selector is implemented before `validate` (plan P2.7) because `validate --view` reports empty selections and selection warnings (ADR-0017).

### 9.2. Family projectors

Each projector maps the selection to one of three disjoint IRs:

- Architecture IR: nodes, nested boundary groups, typed edges and portable layout hints.
- Flow IR: selected nodes, directed/bidirectional semantic flows, labels, explicit payload and protocol metadata. A selected undirected dependency is an FFV error, not a warning or omitted edge. Stores and processing/messaging nodes are distinguished by semantic shape and icon; source/sink annotations are post-MVP.
- Sequence IR: ordered participants, messages (including self-messages) and notes; groups and activation spans are post-MVP. Messages use a solid line and one from-to arrowhead without timing semantics; responses are separate reverse messages and relationId does not import relation appearance. Optional legends explain only message/note/numbering constructs.

Deprecated elements retain their `[deprecated]` text marker in labels and accessible descriptions in every family. Architecture/flow `display.relationLabels` hides only user labels; mandatory semantic labels remain visible. All label text is composed by `generation/labels.py` according to PRD §13.10 and ADR-0011; projectors carry the parts, not composed strings.

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

- write a generated-file header with FlowFrame version and resolved theme fingerprint equal to the manifest's `theme.sha256`, computed by the same aggregate algorithm (not a D2 content hash or authenticity token). The header is exactly these three comment lines at the start of the file:

  ```text
  # Generated by FlowFrame <version>. Do not edit.
  # flowframe-header: v1
  # flowframe-theme-sha256: <64 lowercase hex digits>
  ```

  A missing or malformed header and a fingerprint mismatch in `render` are validation errors (1),
- emit only constructs owned by the relevant family generator,
- escape IDs, labels and string values in one audited module,
- serialize properties in a fixed order,
- serialize nodes before edges,
- preserve IR order and use stable ID tie-breakers,
- resolve all visual values through theme tokens and framework mappings, applying the shape and token application tables of PRD §13.9 (ADR-0012) and line styles from `generation/line_styles.py` (ADR-0015),
- compose labels with `generation/labels.py`, wrapping to the framework `label-max-width` with the shared metrics module (ADR-0007, ADR-0011),
- avoid timestamps, absolute paths and process-specific values,
- finish files with exactly one newline.

Raw D2 snippets from YAML are forbidden. Unknown tokens, kinds or semantics are validation errors rather than pass-through content.

Generated identifiers MUST prefix source IDs in a reserved internal namespace and escape all D2 syntax. D2 keywords remain valid source IDs; source-to-generated mapping is injective. Derived IDs use separate namespaces and stable semantic identity, never random UUIDs or Python hashes.

Generated D2 is self-contained: classes are materialized from resolved theme tokens/mappings and internal templates, with approved generic icons embedded as bounded SVG data URIs. Icons are sanitized with the logo profile at package build; at runtime their SHA-256 is verified against `index.json` (ADR-0013). There are no output imports, remote URLs or filesystem asset paths. The v0.1 generic index maps kinds, not free-text technologies, to assets with hashes/licenses. Deduplicate embedded URI declarations by sanitized asset SHA-256, using reusable classes/declarations so each unique asset appears once in D2 regardless of node count. An explicit null entry is a silent semantic-shape fallback; missing entries, missing declared files or hash mismatches are invalid-pack errors. P0 verifies reuse with the pinned D2, including interactions with per-kind styling.

Required D2 lint rules reject missing/invalid generated headers, imports, absolute or relative filesystem references, remote URLs, raw source-injected constructs, markdown/HTML labels, and unapproved data URIs. Only bounded SVG icon data URIs whose decoded bytes match an icon-pack index hash are allowed. The lint validates content, not just a trusted-looking header.

The generated D2 is an auditable intermediate artifact. Byte-for-byte golden tests are the main regression boundary between projection and external rendering.

---

## 11. D2 process boundary

The renderer wrapper invokes D2 without a shell and with:

- an explicit executable path,
- an explicit layout engine,
- pinned or recorded flags,
- an allowlisted environment,
- a bounded working directory inside the build workspace,
- the fixed timeout from `resources/limits/v1.json` (ADR-0009; not user-configurable in v0.1),
- bounded captured stdout and stderr,
- separate validation and render steps where supported,
- a new process group.

The child environment is built from an allowlist, not inherited: a minimal `PATH`, `HOME` and `TMPDIR` pointing into the build workspace, `LC_ALL=C.UTF-8`, `TZ=UTC`, and only the `D2_*` variables FlowFrame sets itself. Network access is not required and remote asset references are already rejected before this stage.

On SIGINT or timeout the wrapper sends SIGTERM to the process group, waits a fixed grace period, then sends SIGKILL, removes staging owned by the run and returns 130 (SIGINT) or 4 (timeout). No child process may outlive the command.

The wrapper MUST record the verified executable SHA-256, reported D2 version, layout engine and version if exposed, arguments and normalized failure details. A timeout or non-zero exit is an error even if a partial SVG exists. Partial outputs MUST be removed or left only in a diagnostic temporary directory, never published as successful artifacts.

`d2 validate` is the intended syntax boundary for render/build. Stage 0 MUST verify the pinned release's command and exits and retain local version-specific observations in its ADR. `d2 fmt --check` is not a substitute. Compile does not invoke D2. A syntax rejection for freshly generated build input is FFX/internal (6); rejection of a separately supplied render artifact is FFD/validation (1). Classify a crash/timeout as execution (4), not a syntax rejection, and retain bounded process evidence for diagnostics.

### 11.1. Artifact publication

Individual compile/render outputs use a sibling temporary file and `os.replace` after successful verification. Before writing, the output path is compared with every input path of the invocation, and an existing file is replaced only if it is a FlowFrame artifact of the same type (valid D2 header, or the SVG marker comment of ADR-0008); otherwise the command fails with policy error 5 and leaves the file unchanged. Complete builds publish to an ordinary dedicated output directory, without symlinks or generation history. All decisions in this section are recorded in [ADR-0010](adrs/0010-output-publication.md).

For target `<parent>/<name>`, private control state lives in `<parent>/.<name>.flowframe/`: one stable advisory-lock file, one staging directory, at most one previous-output directory and a small recovery journal. It is the only path outside the target a build writes, as permitted by PRD §16.4. Documentation marks this control directory as untracked build state; it is not part of the three public artifacts.

Before step 1, reject (policy, 5) an output directory that is a filesystem root, the process working directory, the resolved project root, an ancestor of any resolved source, configuration, theme or asset file, inside a theme directory, or a symlink. Supported filesystems are local POSIX filesystems; network filesystems are unsupported and not detected in v0.1. Staging and target MUST be on the same filesystem. Existing control state must have valid ownership records for this exact target; a name alone never authorizes recursive cleanup.

1. Canonicalize the parent and target identity, reject an output symlink and acquire the per-target OS advisory lock. A competing writer fails immediately with a stable FFR execution diagnostic (4). Do not delete the lock file to break a held lock.
2. Recover an interrupted prior publication before starting work. If the target is missing and the recorded intact backup exists, restore it. If the target contains the verified newly published set, recognize completion. Do not replace unrelated content encountered at the target or control paths; report the conflict and preserve the backup.
3. Accept an absent target, an empty ordinary directory or a previous FlowFrame output whose manifest (any manifest version this release can read, ADR-0004), exact file inventory and output hashes validate. Reject unrelated files or modified artifacts with guidance to choose a dedicated directory. Never claim ownership from filenames alone.
4. Prepare D2/SVG in the single staging directory, write the manifest last, verify the complete artifact set and flush it before publication. Failure here leaves the old target unchanged. Remove only staging state owned by this run.
5. Before replacement, clean the older retained backup if it is verified as FlowFrame-owned; on cleanup failure stop before touching the current target. Persist recovery intent, rename the old target to the backup slot (if one exists), then rename staging to the target. Record completion. First publication needs only the second rename; an accepted empty target can occupy the temporary backup slot.
6. On failure/SIGINT during replacement, attempt to restore the recorded backup when no new target was published. Report an unsuccessful rollback and the backup/recovery location explicitly. A crash is resolved by the same journal-aware recovery on the next invocation.
7. Retain at most the current output, one previous output and one in-progress staging set. Automatically remove obsolete owned state under the lock; no unbounded accumulation or cleanup command is required. An empty backup can be removed after success.

Two renames are not an atomic directory swap: readers may briefly see no target, and separate concurrent file opens have no snapshot guarantee. Consumers should read/copy/commit ordinary output files after a successful build, rather than reading through a publication in progress. Validation/render failure before publication preserves the old output; rollback after a publication failure is best effort, with explicit recovery status, not an unconditional durability promise. Output I/O, lock conflicts and failed recovery use execution category/code 4; unsafe path redirection remains policy/code 5.

P0.7 tests rename failure, crash windows and ownership checks on APFS and ext4. P3.8 implements this bounded state machine; no optional symlink mode, content-addressed generations, exported copies or whole-project transaction is in scope.

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

The grammar is normative in PRD §13.2 and is not repeated here. Parsing produces a small token list (`Literal` and `Source`) during configuration validation. Rendering never performs ad hoc string replacement.

### 12.3. Decoration layout

Decorations are composed after D2 renders the semantic body:

1. parse and verify the body SVG,
2. derive its view box and measurable body bounds,
3. wrap all decoration text according to the PRD, then render title and subtitle as one title block using pinned theme fonts,
4. sanitize, canonicalize and measure the selected logo; require a finite intrinsic view box,
5. scale the logo proportionally, up or down, to the largest size fitting its per-asset `maxWidth` × `maxHeight` display box ("contain"),
6. render the footer fragment and the legend (ADR-0015),
7. compute the top band as the maximum occupied top-slot height plus `decoration-gap` above and below,
8. compute separate stacked legend/footer bands, each with `decoration-gap` above and below,
9. calculate slot-aware canvas width using the exact PRD §13.6 formula (including symmetric clearance for a centered title),
10. translate the semantic body without changing its internal geometry,
11. place decorations into their configured slots and embed the sanitized logo nodes,
12. inject accessibility metadata (ADR-0014),
13. canonicalize and verify the complete SVG (ADR-0008).

Slots and slot conflicts are normative in PRD §13.2 and §13.6; conflicts are evaluated after built-in defaults are applied. An absent optional decoration occupies no band. Decoration spacing is deterministic and comes from the theme, not from View metadata.

Text measurement MUST use shipped or project-approved pinned fonts through `presentation/text_metrics.py`, shared by the compositor and label wrapping. Host-dependent browser measurement is not acceptable. Glyph coverage is not validated in v0.1; the covered scripts are documented.

Deterministic measurement does not by itself guarantee identical consumer rendering. [ADR-0007](adrs/0007-svg-text-and-fonts.md) MUST select one output-wide text policy in P0.4:

- preserve accessible/searchable SVG `<text>` and pin or embed fonts, accepting documented rasterization differences between consumers, or
- convert visible text to deterministic paths, accepting larger output and loss of native selection/search while providing per-object accessible names/descriptions plus a generated textual summary.

The policy applies to D2 body text and FlowFrame decorations. Path conversion cannot be selected unless the accessibility baseline remains satisfied. A hybrid policy is allowed only if the ADR defines a clear boundary and tests both portability and accessibility; it MUST NOT arise accidentally from two independent render paths.

Sanitized-logo hashes cover the canonical standalone asset. During embedding, the compositor MUST deterministically rename any asset IDs that collide with body or decoration IDs and update all fragment references. Collision handling MUST NOT mutate the stored standalone sanitized bytes or their manifest hash. Generated title and logo accessibility nodes use the same collision-safe ID allocator.

The final `build` SVG MUST expose deterministic `title`/`desc` and ARIA references (`role="img"`, `aria-labelledby`, `aria-describedby`) as defined in [ADR-0014](adrs/0014-svg-accessibility-metadata.md). Visible title text and the theme asset `alt` value remain escaped plain text. Always emit a top-level `<desc>` using the ADR-0014 template and per-object `<title>`/`<desc>` located through the hook mechanism proven in P0.3. A missing visible title uses the View ID as its accessible name. Legends describe used non-color relation appearances (ADR-0015), use pinned theme text, and occupy a measured bottom band above the footer. Their space is included before final sizing. `render` body SVG carries none of this metadata.

PRD §13.3 defines numeric token units, required colors, default mappings and typography; PRD §13.9 defines token application and the contrast pairs checked by `presentation/contrast.py`. P0.4 verifies the required custom TTF faces with D2; fonts used by body and compositor are pinned consistently. Title/subtitle/footer text is NFC-normalized, measured and greedily wrapped at whitespace, with grapheme-boundary fallback, under `decoration-text-max-width`. Body bounds include the pinned D2 `--pad`; do not crop that padding again. Blocks in a top band are vertically centered, and horizontal edges respect `decoration-padding-x`.

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

The manifest records profile ID, sanitizer implementation/version, original SHA-256 and sanitized SHA-256 for each used Theme/Brand Pack asset (`kind: logo`). Identical bytes, profile and implementation version MUST produce byte-identical sanitized output.

The same sanitizer and profile process the generic icon pack when the FlowFrame package is built; the committed icon files are canonical sanitized bytes and are only hash-verified at runtime (ADR-0013).

---

## 14. SVG normalization and verification

The published SVG is canonical: normalized, verified and then hashed; snapshot tests compare published bytes ([ADR-0008](adrs/0008-canonical-svg-output.md)). Raw renderer SVG is kept only in a diagnostic temporary directory with `--debug`. The first child node of the root is the marker comment `<!-- Generated by FlowFrame. Do not edit. -->`.

The final SVG verifier MUST ensure:

- one root `svg` element with a valid finite view box,
- no scripts, events, `foreignObject`, external URLs or undeclared namespaces,
- no remote font or image references; permit only hash-verified bounded embedded icon SVGs and renderer-emitted font data with expected MIME types and a documented source-font-to-output transformation,
- renderer `<style>` content restricted to the ADR-0008 CSS allowlist, parsed with `tinycss2`,
- all fragment references resolve,
- title/logo/footer bounds remain inside the final canvas,
- the semantic body is not clipped by decoration bands,
- accessibility references resolve and visible metadata remains plain text,
- numeric values are finite and within configured canvas limits.

Normalization MUST make output stable by fixing namespace prefixes, lossless numeric serialization, attribute order and explicitly ignorable metadata, and MUST be idempotent. It MUST NOT reorder visible child elements when painter order affects appearance. D2 may subset or re-encode fonts: source TTF hashes and embedded-font hashes need not be equal. P0 MUST characterize that pinned transformation; verification checks allowed MIME/content, bounds and provenance, and records both source and embedded hashes rather than demanding byte identity with the source TTF.

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
- generated D2 and canonical SVG hashes and their manifest-relative paths,
- optional `generatedAt` only when `SOURCE_DATE_EPOCH` supplies a valid nonnegative epoch, serialized as UTC RFC 3339 with `Z`.

Source paths are relative to the project root; output paths are relative to the manifest directory. Built-in resources use package resource IDs; absolute host paths are forbidden. Renderer flags record logical options/resource IDs instead of resolved absolute font paths. Every used icon/font records kind, ID, project path or package ID/version, license reference and SHA-256 in a `resources` array. `assetProcessing` is present exactly when Theme/Brand Pack logos are sanitized during the build, each entry with `kind: logo`; pre-sanitized generic icons appear only in `resources` (`kind: icon`); font transformations are recorded with source/embedded hashes in `resources`, not as logo-sanitizer runs. Map keys and arrays with set semantics are serialized in stable order. JSON uses UTF-8, fixed indentation and one trailing newline.

No wall-clock timestamp is generated by default. With identical `SOURCE_DATE_EPOCH` and inputs the entire manifest is stable; no invented JSON Schema `volatile` keyword is needed. Reproducibility tests compare all fields under the same environment. Input hashes cover raw bytes even when parsing normalizes BOM/NFC.

---

## 16. CLI behavior

The exact signatures, defaults, category-to-exit mapping and precedence are defined once in [PRD §17](prd.md#17-cli-contract); stage coverage per command is defined in [ADR-0017](adrs/0017-command-stage-matrix.md). CLI help and contract tests MUST follow them.

- `validate` supports model-only, model+view and exclusive theme-only forms; with a View it runs selection and IR invariants; it never invokes D2. `validate --all` is post-MVP.
- `compile` validates and emits self-contained D2 only. It requires no D2 executable and applies single-file overwrite protection (§11.1).
- `render` verifies the generated header, theme fingerprint and full D2 lint, then runs D2 validation/rendering. It emits canonical body SVG only. An explicit config is used for custom fonts/theme and `render.layoutEngine`; without it only built-in configuration is used.
- `build` composes the complete branded diagram, adds accessible summary/legend and publishes one complete artifact set using recoverable ordinary-directory replacement. Layout precedence is `--layout` > `render.layoutEngine` > `elk` (ADR-0003); only ELK is accepted in v0.1 and no fallback exists.
- `build --all` uses the explicit project `views` registry and a single supplied model. It runs every validation stage up to the family IR for all views before rendering the first one, then renders in View-ID order and publishes independently to `DIR/<view-id>/` using section 11.1 (no concurrent-reader atomicity). A render or publication failure stops processing and reports already published views.
- `review` selects stages of the same validation service as `validate`, without duplicate rules or D2/AI. Default modes produce the same findings/status as model+view validate. P7 source-conformance uses explicitly supplied sources and an optional adapter and reports `FFA` warnings (ADR-0025); visual review is post-MVP.
- `compare-layouts` remains post-MVP, compiling once and rendering with at least two explicitly selected engines without fallback.

All processing commands support `--format text|json` and `--debug`. JSON diagnostics are one `flowframe-diagnostics/v1` array on stderr and stdout stays empty; optional redacted debug objects are contained within diagnostic entries, never emitted as extra text. Rich/Typer styling and Python warnings MUST NOT reach either stream in JSON mode. No implicit warning promotion is implemented.

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

No stability guarantee is made for lower-level Python modules in v0.1. The CLI `render` and `review` commands use internal services; facade entry points for them are post-MVP.

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

- schema positive/negative fixtures, including agreement of schema `default` annotations with `defaults/v1.json`,
- loader resource-limit and source-location tests, YAML 1.2 scalars (`yes/no/on/off`), merge keys, custom tags, BOM and NFD input,
- label composition per family and display-flag combination,
- property tests: detail and audience never change selection; warnings never change generated D2; D2 ID mapping is injective and D2 keywords remain valid IDs,
- contrast validator pairs for built-in and custom themes,
- theme hash test vector (ADR-0020),
- diagnostics JSON schema validation and registry completeness,
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
- deterministic repeated builds in different absolute paths (D2 and canonical SVG),
- global branding consistency across every family,
- decoration sizing and slot-conflict cases, including conflicts introduced by defaults,
- differential test: `validate`, `review` and `build` report identical diagnostics up to the IR stage,
- `compile` without D2 on PATH; `FLOWFRAME_D2` relative, non-executable or directory,
- forbidden output directories and single-file overwrite refusal,
- no orphan D2 process after SIGINT or timeout,
- JSON-mode stream purity with `--debug` and `PYTHONWARNINGS=always`.

### 19.3. Snapshot and security tests

- normalized SVG snapshots for the golden corpus,
- bounds checks for long and Unicode labels,
- title/subtitle/footer/logo combinations,
- gradients, clips, masks and supported bounded filters,
- XXE, external URL, event handler, CSS import and filter-bomb rejection,
- path traversal and symlink escape attempts,
- fuzz/property tests for footer parsing, D2 escaping and SVG references.

Visual snapshots require intentional approval by the Accessibility and design reviewer (ADR-0021). A snapshot update alone is not evidence that a visual regression is acceptable.

---

## 20. Performance and observability

Stage 0 MUST record baseline timings and memory for small and medium golden diagrams. v0.1 performance budgets ([ADR-0023](adrs/0023-performance-budgets.md)) are then added to CI as generous regression limits, not hard real-time guarantees.

Debug logs MAY include stage timings, selected counts, renderer command metadata and cache decisions. Normal mode remains concise. Logs MUST never include unredacted environment variables or source contents by default.

Caching is optional for v0.1. If introduced, its key MUST include all source hashes, FlowFrame version, schema versions, resolved theme/asset hashes, sanitizer identity, D2/layout identity, flags and platform-relevant font identity. An incomplete cache key is worse than no cache.

---

## 21. Compatibility and versioning

Schema version changes follow [ADR-0004](adrs/0004-schema-authority-and-contract-versioning.md):

- closed contracts require a new `schemaVersion` for any newly accepted field or enum value, including optional additions; a version covers the accepted language, not merely required fields,
- changed meaning, removed fields or newly required fields require a new contract version,
- a release reads the current and the immediately previous version of each source contract; v0.1 reads only v1,
- FlowFrame MUST reject unsupported contract versions with a migration-oriented diagnostic,
- generated artifacts record both contract and tool versions,
- automatic migration is post-MVP; v0.1 may provide only guidance.

Theme packs declare their own version independently of FlowFrame. The sanitizer profile and sanitizer implementation version are both recorded because a profile-compatible bug fix can still affect canonical bytes.

---

## 22. Architecture decisions

All decisions, including those still open, are tracked in the ADR register [`docs/adrs/README.md`](adrs/README.md). Implementation MUST NOT guess a value that an ADR lists as Proposed or as an open parameter; the dependent work waits for the owning phase. Each ADR records context, tested alternatives, decision and consequences. The actionable order and acceptance gates are defined in [`implementation-plan.md`](implementation-plan.md).

Decisions still open at the time of writing:

| ADR | Open item | Phase |
|---|---|---|
| 0006 | D2 version, per-platform hashes, `d2 validate` behavior | P0.2 |
| 0003, 0011, 0012, 0013, 0014 | profile mapping, `label-max-width`, shape rendering, icon reuse, accessibility hooks | P0.3 |
| 0007 | text policy, default font family, measurement library | P0.4 |
| 0008 | ignorable renderer fields, serializer, CSS allowlist | P0.3 |
| 0010 | journal format and fsync points | P0.7 |
| 0009, 0013, 0019, 0021, 0023 | numeric limits, icon set, display-default verification, snapshot sets and consumer SVG policy, performance budgets | P0.6 |
| 0022 | supported consumers | P0.1 |
| 0025 | AI platform (P7.1), thresholds and corpus (P7.4) | P7 |

Dark theme is deliberately absent because it is post-MVP, consistent with the PRD and P9.
