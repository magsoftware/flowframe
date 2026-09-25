# FlowFrame — Product Requirements and Technical Specification

## 1. Document status

| Field | Value |
|---|---|
| Product | FlowFrame |
| Target release | v0.1 MVP |
| Status | Draft for implementation |
| Primary output | SVG |
| Baseline layout engine | ELK |
| Optional layout engine | TALA |

The keywords **MUST**, **SHOULD** and **MAY** describe mandatory, recommended and optional requirements.

---

## 2. Product goal

FlowFrame is a lightweight, portable diagram-as-code framework for generating consistent IT diagrams from structured semantic models.

It targets three diagram families:

- **Architecture** — infrastructure, deployment, application architecture, system context and network/security views,
- **Flow** — data, integration, event and dependency flows,
- **Sequence** — ordered interactions between system participants.

FlowFrame separates four concerns:

1. the system being described,
2. the purpose and scope of a particular view,
3. projection of that view into a diagram-family-specific representation,
4. visual rendering.

The framework, rather than an AI model or diagram author, controls visual style. AI integration is optional and produces structured input, not final styling.

### 2.1. Problem statement

Architecture diagrams created manually or generated directly by AI commonly suffer from:

- inconsistent colors, shapes and naming,
- excessive or missing detail,
- poor repeatability between generations,
- unclear boundaries and relation semantics,
- output that is difficult to review in Git,
- coupling to a specific vendor or drawing tool,
- manual layout work after every architecture change.

FlowFrame addresses these problems with validated schemas, deterministic code generation, a centralized design system and automatic layout.

### 2.2. Target users

- architects documenting systems and infrastructure,
- developers explaining integrations and runtime behavior,
- platform and security engineers documenting boundaries and controls,
- technical writers embedding diagrams in documentation,
- CI/CD pipelines validating and rendering diagrams,
- AI agents converting descriptions into structured diagram models.

---

## 3. Scope

### 3.1. MVP scope

FlowFrame v0.1 MUST support:

- YAML input with JSON Schema validation,
- a reusable System Model,
- a View Specification selecting and presenting part of that model,
- three initial view types:
  - `architecture/infrastructure`,
  - `flow/integration-flow`,
  - `sequence/sequence`,
- deterministic generation of D2 source,
- SVG rendering through a pinned D2 CLI,
- ELK as the portable baseline layout engine,
- a centralized light theme,
- a small, documented semantic vocabulary,
- semantic validation in addition to schema validation,
- local and CI execution without network access,
- example models and golden-file tests,
- one AI integration adapter that generates System Model and View Specification files.

### 3.2. Post-MVP scope

- remaining architecture and flow subtypes,
- dark theme,
- optional TALA integration,
- vendor icon packs,
- PNG, PDF and PPTX outputs,
- advanced visual review using rendered images,
- additional AI platform adapters,
- model migration tooling.

### 3.3. Non-goals

The MVP MUST NOT attempt to build:

- a custom diagram DSL,
- a drag-and-drop editor or custom GUI,
- a complete enterprise architecture repository,
- a live infrastructure discovery system,
- a component database or CMDB,
- a complete catalog of cloud services,
- bidirectional synchronization between YAML and hand-edited D2,
- pixel-perfect manual placement,
- automatic verification that a model matches the real deployed system.

---

## 4. Core principles

FlowFrame SHOULD be:

- lightweight and usable from a CLI,
- portable across local environments and CI/CD,
- vendor-neutral at the semantic-model level,
- deterministic for the same inputs and pinned toolchain,
- easy to review and maintain in Git,
- semantics-driven rather than style-driven,
- strict at system boundaries and extensible through versioned schemas,
- usable without AI,
- offline by default during validation and rendering,
- accessible without relying on color alone.

AI MUST NOT define raw colors, fonts, line weights, arbitrary shapes or manual coordinates.

AI MAY describe:

- component meaning and technology,
- boundaries and containment,
- relations and interaction properties,
- scenario messages,
- information priority,
- diagram intent and audience,
- portable layout preferences exposed by FlowFrame.

---

## 5. Source of truth and artifacts

For v0.1, the source of truth is:

```text
system-model.yaml + view.yaml
```

The following files are generated artifacts:

```text
diagram.d2
diagram.svg
manifest.json
```

Generated D2 MUST contain a header stating that it must not be edited manually. Rebuilding a diagram MAY overwrite generated artifacts.

AI MUST produce or update only structured source files. The deterministic compiler is solely responsible for D2 generation and visual styling.

Hand-authored D2 MAY be accepted in a future compatibility mode, but that mode will not receive the same semantic and styling guarantees.

---

## 6. Technology decisions

### 6.1. Diagram language: D2

D2 is the rendering target because it provides:

- readable text syntax,
- SVG generation,
- reusable classes and variables,
- imports and model-view composition,
- icons and containers,
- sequence diagrams,
- multiple layout engines,
- suitable primitives for software architecture diagrams.

D2 is an implementation dependency, not the public FlowFrame authoring format. FlowFrame schemas MUST remain independent enough to allow another renderer in the future.

### 6.2. Layout engines

**ELK is the baseline and default layout engine.** It is bundled with D2, works well for hierarchical diagrams and does not require a separate commercial license.

**TALA is optional.** It may produce better results for non-hierarchical architecture diagrams, but it is separately installed, closed-source, licensed for commercial use and can produce different layouts after small input changes.

The framework MUST NOT silently switch layout engines based on a subjective assessment of visual quality.

Fallback behavior is limited to:

- failing with a clear diagnostic when the selected engine is unavailable, or
- using ELK only when the user explicitly enables an `allow-engine-fallback` option.

A separate comparison command MAY render the same view with all installed engines.

### 6.3. MVP implementation

The recommended MVP implementation is a small Python 3.12 CLI that:

- parses YAML,
- validates it against JSON Schema,
- performs semantic validation,
- projects a view into a family-specific intermediate representation,
- generates D2,
- invokes a pinned D2 CLI,
- records tool versions and options in a manifest.

JSON Schema is the public validation contract. Python model classes MAY provide additional internal typing but MUST NOT replace the published schemas.

Direct integration with the D2 Go library is deferred until process startup or CLI limitations justify the additional coupling.

---

## 7. System architecture

```text
Natural language / source documentation
                 │
                 ▼
          optional AI adapter
                 │
                 ▼
          System Model + View
                 │
        ┌────────┴────────┐
        │                 │
  schema validation  semantic validation
        │                 │
        └────────┬────────┘
                 ▼
          view projection
                 │
                 ▼
       family-specific normalized IR
                 │
                 ▼
        deterministic D2 generator
                 │
                 ▼
             diagram.d2
                 │
          d2 validate + render
                 │
                 ▼
      diagram.svg + manifest.json
                 │
                 ▼
        lint / snapshot / review
```

### 7.1. Required separation of responsibilities

- **System Model** describes reusable facts about the system.
- **View Specification** describes what to show and for whom.
- **Projection** selects, aggregates and normalizes information for a diagram family.
- **Family IR** captures information unique to architecture, flow or sequence diagrams.
- **D2 generator** maps the normalized IR to framework-owned classes and templates.
- **Renderer** validates and renders D2 without changing system semantics.
- **Reviewer** reports problems; it MUST NOT silently alter the System Model.

---

## 8. System Model

### 8.1. General rules

The System Model MUST:

- declare a schema version,
- use stable and unique identifiers,
- distinguish elements from boundaries,
- support containment,
- use references rather than duplicating elements,
- keep rendering properties out of the model,
- allow technology-specific metadata without changing semantic kinds,
- support ordered scenarios used by sequence views.

The MVP supports a single primary containment parent per element. Other classifications SHOULD be represented with tags. Overlapping visual boundaries are deferred until their rendering semantics are defined.

### 8.2. Example

```yaml
schemaVersion: flowframe/v1
system:
  id: payments-platform
  label: Payments Platform

boundaries:
  - id: production
    kind: environment
    label: Production

  - id: application-network
    kind: network
    label: Application Network
    parentId: production

  - id: data-network
    kind: network
    label: Data Network
    parentId: production

elements:
  - id: app-gateway
    kind: gateway
    label: Application Gateway
    technology: Azure Application Gateway
    parentId: application-network
    tags: [entrypoint, runtime]

  - id: api
    kind: service
    label: Backend API
    technology: Kubernetes
    parentId: application-network
    tags: [runtime]

  - id: postgres
    kind: database
    label: PostgreSQL
    technology: Azure Database for PostgreSQL
    parentId: data-network
    tags: [data, runtime]

relations:
  - id: gateway-calls-api
    source: app-gateway
    target: api
    semantic: request
    interaction: synchronous
    protocol: HTTPS

  - id: api-reads-postgres
    source: api
    target: postgres
    semantic: data-access
    interaction: synchronous
    protocol: PostgreSQL

scenarios:
  - id: submit-payment
    label: Submit payment
    steps:
      - id: request-payment
        from: app-gateway
        to: api
        kind: message
        label: POST /payments
        protocol: HTTPS

      - id: persist-payment
        from: api
        to: postgres
        kind: message
        label: Insert payment
```

### 8.3. Identifier rules

- IDs MUST be unique within the model.
- IDs MUST remain stable when labels change.
- IDs MUST use lowercase ASCII letters, digits and hyphens.
- Relations and scenario steps SHOULD have explicit IDs to support diagnostics and traceability.
- Labels are human-readable and MAY contain Unicode.

---

## 9. Controlled vocabulary

### 9.1. Element kinds

The MVP vocabulary is intentionally small. The definitions are normative:

| Kind | Meaning |
|---|---|
| `actor` | A human or organizational participant interacting with the system. |
| `external-system` | A software system outside the modeled system's ownership boundary and shown as a whole. |
| `web-application` | A user-facing software element whose primary interface is a web UI. A separately deployed backend is modeled as a `service`. |
| `service` | An independently addressable software capability or API. |
| `worker` | A non-interactive compute process triggered by a job, schedule or event. |
| `gateway` | An ingress, proxy or routing element mediating traffic to other elements. |
| `database` | A persistent structured data store with query or transaction semantics. |
| `cache` | A temporary or derived data store primarily used to reduce access latency. |
| `queue` | A messaging element that buffers or distributes asynchronous messages. |
| `storage` | Persistent object, blob or file storage without database semantics. |
| `identity-provider` | A component that authenticates identities or issues identity assertions or tokens. |
| `secret-store` | A component that protects and provides secrets, keys or certificates. |
| `security-control` | A component that enforces or observes a security policy and has no more specific semantic kind. |

Vendor products MUST use a semantic kind plus an optional `technology` value:

```text
Azure Service Bus              → kind: queue
Azure Database for PostgreSQL  → kind: database
Azure Key Vault                → kind: secret-store
AWS WAF                        → kind: security-control
```

Adding a vendor product MUST NOT require adding a new semantic kind.

### 9.2. Boundary kinds

| Kind | Meaning |
|---|---|
| `subsystem` | A logical system contained within the root system or another subsystem. |
| `environment` | A lifecycle or deployment environment such as development, staging or production. |
| `network` | A network segment, virtual network or subnet. |
| `trust-zone` | A region governed by a shared trust level or security policy. |
| `cluster` | A compute or orchestration cluster. |
| `namespace` | A logical isolation scope within a cluster or platform. |

The top-level `system` object represents the modeled system and provides its outer boundary when a view displays it. Nested logical systems use the unambiguous `subsystem` boundary kind.

Boundaries define containment or a visible grouping. They are not runtime components and MUST NOT use component styling.

### 9.3. Relation properties

Relation semantics are expressed using orthogonal properties rather than one overloaded type.

`semantic` identifies meaning:

| Value | Meaning |
|---|---|
| `request` | A directed invocation or command sent to another element for processing. |
| `event` | A notification that a fact or state change occurred, normally delivered asynchronously. |
| `data-access` | A read, write or query performed against a data store. |
| `data-flow` | Transfer of a data set, stream or artifact between elements. |
| `dependency` | A structural or runtime dependency that does not necessarily represent network traffic. |
| `replication` | Copying or synchronizing state between stores or instances. |
| `control` | A management, scheduling or orchestration signal. |
| `authentication` | Proof or verification of identity, including token or assertion exchange. |
| `authorization` | A permission or policy decision about an attempted action. |

`interaction` identifies timing:

| Value | Meaning |
|---|---|
| `synchronous` | The source waits for completion or an immediate response before continuing the modeled interaction. |
| `asynchronous` | The source does not wait for the target to complete processing. |
| `not-applicable` | Timing semantics do not apply, for example to a purely structural dependency. |

Optional properties include:

```text
protocol
label
directionality
technology
encrypted
tags
```

`source` and `target` are required for every relation, including an `undirected` relation. They identify the two endpoints and provide stable serialization and traceability. For `directed` and `bidirectional` relations, their order defines the normal forward order; reverse flow is represented by swapping them, not by another enum value. For `undirected`, their order has no flow meaning. `directionality` MAY be `directed`, `bidirectional` or `undirected`, defaults to `directed` and controls the directional interpretation and arrowhead rendering without changing endpoint requirements.

`encrypted` is a boolean describing transport encryption for that relation. It does not assert broader end-to-end or at-rest encryption.

`internet` is not a relation type. It SHOULD be modeled as a network boundary, an external network element or view metadata describing the route.

`trust` is not a generic connection type. Trust boundaries and authentication/authorization relations MUST be modeled explicitly.

### 9.4. Scenario step kinds

The v0.1 scenario vocabulary is:

| Kind | Required fields | Meaning |
|---|---|---|
| `message` | `id`, `from`, `to`, `label` | An ordered interaction between two participants. A self-message uses the same ID in `from` and `to`; it is not a separate kind. |
| `note` | `id`, `participant`, `label` | An explanatory note attached to one participant at that point in the scenario. `participant` MUST reference an element ID; boundary IDs are invalid. |

Activation spans and grouped fragments such as `alt`, `loop` and `parallel` are post-MVP extensions and MUST NOT appear in a v0.1 source model.

---

## 10. View Specification

### 10.1. Purpose

A view selects and projects information from the System Model. It MUST NOT duplicate the complete system definition.

Every view declares:

- schema version,
- stable view ID,
- family and subtype,
- audience and detail level,
- selection criteria,
- display options,
- a portable layout profile,
- optional engine-specific overrides in a clearly isolated section.

### 10.2. Supported families and subtypes

| Family | MVP subtype | Planned subtypes |
|---|---|---|
| Architecture | infrastructure | deployment, application-architecture, system-context, network-security |
| Flow | integration-flow | data-flow, event-flow, dependency-flow |
| Sequence | sequence | — |

### 10.3. Audience and detail

`audience` is a required enum in v0.1:

| Value | Intended reader |
|---|---|
| `architect` | System and solution architects evaluating structure and trade-offs. |
| `developer` | Engineers implementing or debugging software behavior. |
| `platform` | Engineers operating runtime platforms, networks and deployment infrastructure. |
| `security` | Engineers reviewing trust boundaries, identity and security controls. |
| `operations` | Engineers operating and supporting the deployed system. |
| `business` | Non-technical stakeholders interested in capabilities and external interactions. |
| `mixed` | A cross-functional audience for which no single specialist profile is appropriate. |

`detail` is a required enum and supplies family-specific display defaults only:

| Value | Default intent |
|---|---|
| `low` | Names, major boundaries and primary relations; protocols and technology details hidden. |
| `medium` | Boundaries and relation labels visible; selected technology and protocol details shown when useful. |
| `high` | All supported metadata relevant to the selected family is shown unless explicitly disabled. |

The detail preset MUST NOT add or remove selected elements. Explicit `display` values override preset defaults. Audience is metadata used by templates, review rules and AI guidance; it MUST NOT silently change selection in v0.1. Each family template MUST document its exact defaults for every detail level.

### 10.4. Architecture view example

```yaml
schemaVersion: flowframe/v1
id: infrastructure-overview
family: architecture
subtype: infrastructure
audience: architect
detail: medium

select:
  tags: [runtime]
  includeRelated: true
  relatedDepth: 1

exclude:
  kinds: []
  tags: [internal-detail]

display:
  boundaries: [environment, network]
  protocols: true
  technologies: true
  relationLabels: true

layout:
  profile: hierarchical
  direction: right
```

### 10.5. Sequence view example

```yaml
schemaVersion: flowframe/v1
id: submit-payment-sequence
family: sequence
subtype: sequence
audience: developer
detail: medium

scenarioId: submit-payment

display:
  protocols: true
  stepNumbers: true

layout:
  profile: sequence
```

A sequence view references exactly one `scenarioId` in v0.1. Combining scenarios requires separate views; multi-scenario composition is deferred until ordering and presentation semantics are defined.

### 10.6. Selection semantics

The selection contract covers:

- whether criteria are combined using AND or OR,
- how `includeRelated` traverses relations,
- how many traversal hops are allowed,
- what happens to relations with an excluded endpoint,
- how containment ancestors are added,
- how empty boundaries are handled.

For v0.1, selection is deterministic and runs in this order:

1. Build the seed set from `select.ids`, `select.tags` and `select.kinds`. Values within one field use OR; different populated fields use AND. If `select` is omitted, all elements form the seed set.
2. If `includeRelated` is true, traverse relations breadth-first from the seed set and add the opposite endpoints through `relatedDepth` hops. `relatedDepth` defaults to `1`, MUST be a positive integer and is invalid when `includeRelated` is false.
3. Apply `exclude.ids`, `exclude.tags` and `exclude.kinds` to elements. Matching any populated exclusion field removes the element; exclusion takes precedence over selection.
4. Include model relations only when both endpoints remain selected. Relations with an excluded or otherwise missing endpoint are omitted.
5. Add the complete containment-ancestor chain needed for every selected element. `display.boundaries` determines which boundary kinds are visible; descendants of a hidden boundary are promoted to the nearest visible ancestor or the view root.
6. Omit empty visible boundaries.

Containment-depth filtering and aggregation are not supported in v0.1. They require explicit flattening and traceability semantics before being added.

---

## 11. Family-specific intermediate representations

A single normalized model is insufficient for every diagram family. Projection MUST produce one of three internal representations.

### 11.1. Architecture IR

Contains:

- selected elements,
- visible nested boundaries,
- selected relations,
- technology labels and icons,
- architecture-specific layout profile.

### 11.2. Flow IR

Contains:

- producers, processors, stores and consumers,
- ordered or directed flows,
- events or data artifacts where applicable,
- protocol and delivery semantics,
- flow-specific layout profile.

### 11.3. Sequence IR

Contains:

- ordered participants,
- ordered messages,
- self-messages represented as messages whose source and target participant are identical,
- notes.

Activation spans and groups such as `alt`, `loop` and `parallel` may be added after the MVP.

All IR nodes and edges MUST retain their source model IDs for diagnostics and traceability.

---

## 12. Layout model

### 12.1. Portable layout properties

The portable v0.1 API is limited to:

```text
profile: hierarchical | compact | sequence
direction: up | down | left | right
```

`direction` is a FlowFrame value mapped to the D2 direction expected by the selected engine.

### 12.2. Engine capability policy

| Capability | ELK | TALA | FlowFrame policy |
|---|---:|---:|---|
| Global direction | yes | yes | portable |
| Per-container direction | no | yes | optional TALA extension |
| Near another object | no | yes | optional TALA extension |
| Locked `top` and `left` position | no | yes | excluded from MVP authoring model |
| Container width and height | yes | yes | renderer-controlled only |

Engine-specific settings MUST be namespaced, for example:

```yaml
layout:
  profile: hierarchical
  direction: right
  engineOptions:
    tala:
      near:
        legend: api
```

Validation MUST report when a selected engine cannot implement an explicitly requested engine-specific option.

Manual coordinates are outside the MVP contract.

---

## 13. Design system

### 13.1. Ownership

All visual properties MUST be owned by framework libraries and the D2 generator. Models and view specifications MUST NOT contain raw D2 style declarations or RGB/HEX values.

An element class MAY be derived from:

- semantic kind,
- state such as external or deprecated,
- emphasis calculated from the view,
- optional technology icon.

### 13.2. Design tokens

The theme MUST centrally define at least:

```text
background
surface
surface-muted
border

text-primary
text-secondary

primary
secondary

compute
database
messaging
storage
security
external
network

line-request
line-event
line-data
line-dependency
line-control
```

D2 classes and variables SHOULD implement these tokens. Because D2 allows object-level properties to override class values, generated or hand-authored D2 MUST be linted if direct D2 input is ever supported.

### 13.3. Relation appearance

Relations MUST be distinguishable using more than color. The mapping SHOULD use a combination of:

- stroke pattern,
- arrowhead,
- thickness,
- label prefix or text,
- color.

The exact mapping belongs to the versioned theme and MUST NOT be generated by AI.

### 13.4. Accessibility

The v0.1 theme MUST:

- target WCAG AA contrast for text and essential strokes,
- remain understandable in grayscale,
- avoid color as the only semantic signal,
- use readable default font sizes,
- provide visible relation labels where required,
- support a generated textual summary or accessible description.

### 13.5. Icons

```text
icons/
├── generic/
├── azure/
├── aws/
├── kubernetes/
└── security/
```

Rules:

- generic local SVG icons are the MVP baseline,
- remote icon URLs are disabled by default,
- rendering MUST work offline,
- each icon pack MUST include license and attribution metadata,
- vendor logos MUST follow the vendor's trademark rules,
- technology without an approved icon falls back to its semantic shape,
- icon absence MUST NOT change component semantics.

---

## 14. Diagram rules

The `rules/` directory is the normative human-readable companion to machine validation.

```text
rules/
├── visual-guidelines.md
├── layout-rules.md
├── naming-rules.md
├── diagram-types.md
├── accessibility-rules.md
└── ai-generation-rules.md
```

Initial rules:

- prefer a clear primary reading direction,
- group elements by meaningful boundaries,
- keep external systems outside the main system boundary,
- omit information irrelevant to the view's purpose,
- minimize crossing lines where supported by the engine,
- use a small number of relation appearances per diagram,
- label relations when direction or meaning is not obvious,
- show protocols only when they add value for the audience,
- show deployment details only in deployment-oriented views,
- keep component names short and unambiguous,
- preserve source model IDs through generation,
- never encode critical meaning using color alone.

A guideline of 7–9 main elements per level MAY produce a warning, but MUST NOT be a universal validation error. Diagram family, nesting and audience determine acceptable complexity.

---

## 15. Validation

Validation is part of the first vertical slice, not a late implementation phase.

### 15.1. Validation layers

```text
1. YAML parsing
2. JSON Schema validation
3. semantic model validation
4. view selection and capability validation
5. normalized IR validation
6. D2 generation lint
7. d2 validate
8. render exit-status validation
9. optional SVG and visual checks
```

### 15.2. Required semantic checks

- unique and valid IDs,
- valid source, target, parent and scenario references,
- note participants reference elements rather than boundaries,
- no containment cycles,
- allowed element and boundary kinds,
- allowed relation-property combinations,
- allowed scenario step kinds, their required fields and uniquely identified ordered steps,
- view family/subtype compatibility,
- valid selection fields and traversal limits,
- compatibility of layout options with the selected engine,
- no raw visual styles in source models,
- no disallowed remote assets,
- no empty mandatory labels.

### 15.3. Diagnostics

Every diagnostic MUST include:

- stable diagnostic code,
- severity (`error`, `warning` or `info`),
- source file and YAML path,
- human-readable explanation,
- suggested correction when practical.

Example:

```text
FFM102 error model.yaml:relations[2].target
Unknown element or boundary ID "order-db".
```

Warnings MUST NOT change generated semantics automatically.

---

## 16. Rendering, portability and security

### 16.1. Reproducibility

`manifest.json` MUST conform to a versioned schema. Its minimum v1 contract is:

```json
{
  "schemaVersion": "flowframe-manifest/v1",
  "flowframeVersion": "0.1.0",
  "source": {
    "model": { "path": "system-model.yaml", "schemaVersion": "flowframe/v1", "sha256": "..." },
    "view": { "path": "infrastructure-view.yaml", "schemaVersion": "flowframe/v1", "sha256": "..." }
  },
  "renderer": {
    "d2Version": "...",
    "layoutEngine": "elk",
    "layoutEngineVersion": "...",
    "flags": []
  },
  "theme": { "id": "flowframe-light", "version": "1" },
  "outputs": {
    "d2": { "path": "diagram.d2", "sha256": "..." },
    "svg": { "path": "diagram.svg", "sha256": "..." }
  },
  "generatedAt": "RFC-3339 timestamp"
}
```

Paths in the manifest MUST be relative to the build root. Hashes MUST use SHA-256. `layoutEngineVersion` MAY be `null` only when the engine does not expose a version. The generation timestamp appears only in the manifest, not in deterministic D2 source.

The same inputs and pinned toolchain MUST generate byte-identical `diagram.d2`. SVG stability MUST be tested using normalized snapshots because renderer metadata may differ between versions.

TALA builds SHOULD use a fixed seed when the installed version supports it.

### 16.2. Offline operation

The default validation and build path MUST NOT require network access. Fonts, themes and MVP icons MUST be locally available and pinned.

### 16.3. SVG portability

The implementation MUST define and test:

- whether fonts and icons are bundled,
- supported documentation renderers and browsers,
- behavior when an icon cannot be embedded,
- maximum accepted input and output sizes,
- SVG sanitization requirements for publication.

### 16.4. Input and process security

- YAML loaders MUST use a safe mode and MUST NOT construct arbitrary objects.
- File imports MUST be restricted to explicitly allowed project roots.
- Generated paths MUST not escape the output directory.
- Remote resources MUST be disabled unless explicitly allowed.
- D2 execution MUST have a timeout and a bounded output size.
- CI SHOULD run rendering in an isolated environment.
- Untrusted source documentation supplied to AI MUST be treated as data, not as executable instructions.

---

## 17. CLI contract

Proposed commands:

```text
flowframe validate MODEL VIEW
flowframe compile MODEL VIEW --out diagram.d2
flowframe render diagram.d2 --layout elk --out diagram.svg
flowframe build MODEL VIEW --output-dir build/
flowframe compare-layouts MODEL VIEW --output-dir build/layouts/
flowframe review MODEL VIEW
flowframe version
```

### 17.1. Exit codes

```text
0  success
1  validation or compilation error
2  invalid command usage
3  missing external dependency
4  renderer failure or timeout
5  policy or security violation
```

### 17.2. Build output

```text
build/
├── diagram.d2
├── diagram.svg
└── manifest.json
```

The CLI MUST write diagnostics to stderr and MUST NOT infer success from the presence of an output file. It MUST check the D2 process exit status because D2 may leave a partial output after a rendering error.

---

## 18. AI integration

AI is an optional adapter around the deterministic core.

### 18.1. AI generation contract

The AI adapter MUST:

1. identify diagram purpose and audience,
2. extract or update a System Model,
3. create a compatible View Specification,
4. use only schema-defined values,
5. preserve known IDs when editing an existing model,
6. mark uncertainty rather than inventing critical facts,
7. run validation before compilation,
8. return diagnostics when validation fails.

The AI adapter MUST NOT:

- emit raw styling,
- bypass schemas,
- generate the authoritative D2 directly,
- silently change architecture during layout improvement,
- claim that a model matches a real system without a source-conformance check.

### 18.2. Review modes

Review MUST distinguish:

- **syntax review** — YAML, schema and D2 correctness,
- **semantic review** — internal consistency of the model,
- **source-conformance review** — comparison with supplied source material,
- **visual review** — readability of a rendered SVG/PNG,
- **policy review** — compliance with FlowFrame rules.

Without source material, the reviewer cannot determine whether the documented architecture is factually correct.

### 18.3. Improvement loop

An automated improvement loop MAY adjust only View Specification and layout options. It MUST:

- keep the System Model unchanged unless the user explicitly requests a semantic edit,
- produce a diff,
- validate every iteration,
- use a bounded number of iterations,
- stop when no measurable improvement is found.

### 18.4. Prompts and skills

Prompts and platform-specific skills are adapters, not core runtime dependencies.

```text
prompts/
├── generate-model.md
├── generate-view.md
├── review-diagram.md
├── simplify-view.md
└── compare-with-source.md

skills/
├── architecture-diagram/
├── diagram-review/
└── diagram-refactor/
```

The MVP includes one generation skill or equivalent adapter covering the supported v0.1 view types.

---

## 19. User workflows

### 19.1. Model-first workflow

```text
1. Author or generate system-model.yaml.
2. Author or generate a view.yaml.
3. Run flowframe validate.
4. Run flowframe build.
5. Review diagram.svg and the source diff.
6. Commit source files; generated artifacts are committed according to repository policy.
```

### 19.2. Natural-language workflow

Input:

```text
Create an infrastructure view of:
Internet → Application Gateway → Kubernetes API → PostgreSQL.
The API uses Key Vault and Service Bus.
Show network boundaries, protocols and security controls.
```

Process:

```text
description
→ System Model candidate
→ View Specification candidate
→ validate
→ user-visible assumptions and diagnostics
→ deterministic compile
→ render
→ review
```

Later requests such as `show the event flow` create or update a View Specification. They reuse the existing System Model and do not require the system to be described again.

---

## 20. Repository structure

```text
flowframe/
├── README.md
├── LICENSE
├── pyproject.toml
│
├── src/flowframe/
│   ├── cli.py
│   ├── model.py
│   ├── validation/
│   ├── projection/
│   ├── generators/
│   ├── rendering/
│   └── diagnostics.py
│
├── schema/
│   ├── system-model.schema.json
│   ├── view.schema.json
│   └── manifest.schema.json
│
├── lib/
│   ├── theme.d2
│   ├── components.d2
│   ├── connections.d2
│   └── boundaries.d2
│
├── icons/
│   ├── LICENSES.md
│   ├── generic/
│   ├── azure/
│   ├── aws/
│   ├── kubernetes/
│   └── security/
│
├── templates/
│   ├── architecture/
│   ├── flow/
│   └── sequence/
│
├── rules/
│   ├── visual-guidelines.md
│   ├── layout-rules.md
│   ├── naming-rules.md
│   ├── diagram-types.md
│   ├── accessibility-rules.md
│   └── ai-generation-rules.md
│
├── prompts/
├── skills/
├── examples/
│   ├── infrastructure/
│   ├── integration-flow/
│   └── sequence/
│
├── tests/
│   ├── schema/
│   ├── semantic/
│   ├── golden/
│   └── snapshots/
│
└── tools/
    └── install-pinned-d2.sh
```

---

## 21. Implementation plan

### Stage 0 — Decisions and technical spike

Goal: validate the external dependencies and remove architectural uncertainty.

Scope:

- record ADRs for D2, ELK, optional TALA and the Python CLI,
- pin and verify a D2 version,
- compare ELK and TALA on representative diagrams,
- verify sequence-diagram support,
- verify offline icon and font bundling,
- document TALA licensing and installation constraints,
- confirm SVG behavior in target documentation systems.

Exit criteria:

- three representative diagrams render successfully,
- ELK works offline in local and CI-like environments,
- unsupported engine features are documented,
- no licensing question blocks the MVP.

### Stage 1 — Contract-first vertical slice

Scope:

- System Model v1 schema,
- View Specification v1 schema,
- manifest v1 schema,
- YAML parser and safe loading,
- semantic validator and diagnostics,
- infrastructure projection,
- minimal D2 generator,
- ELK SVG rendering,
- CLI `validate` and `build`,
- one end-to-end infrastructure example.

Exit criteria:

```text
system-model.yaml + infrastructure-view.yaml
→ validate
→ diagram.d2
→ diagram.svg + manifest.json
```

### Stage 2 — Design System v1

Scope:

- tokens and theme,
- component, boundary and connection classes,
- generic icon set and license metadata,
- accessibility rules,
- grayscale and contrast tests,
- D2 generation lint.

### Stage 3 — Remaining MVP families

Scope:

- integration-flow projection and examples,
- sequence scenario projection and examples,
- family-specific normalized IRs,
- golden and SVG snapshot tests.

### Stage 4 — AI adapter and evaluation

Scope:

- model generation prompt,
- view generation prompt,
- structured validation-repair loop,
- diagram review prompt,
- one generation skill or equivalent integration,
- fixed evaluation corpus with expected semantic facts.

### Stage 5 — Packaging and CI/CD

Scope:

- pinned D2 installation or container image,
- checksums and dependency provenance,
- CI validation and rendering job,
- release packaging,
- example GitHub Actions and GitLab CI configuration,
- upgrade and compatibility policy.

### Stage 6 — Optional capabilities

Scope:

- TALA adapter and engine-specific options,
- additional diagram subtypes,
- vendor icon packs,
- additional output formats,
- visual analysis loop.

---

## 22. MVP acceptance criteria

FlowFrame v0.1 is accepted when all of the following are true:

1. All example models and views pass schema and semantic validation.
2. Invalid references, containment cycles and unsupported values produce stable, actionable diagnostics.
3. The same source files and pinned toolchain generate byte-identical D2.
4. All MVP examples render offline with ELK in local and CI environments.
5. Generated D2 contains no raw styles originating from System Model or View Specification files.
6. Infrastructure, integration-flow and sequence examples are produced from the same reusable System Model where applicable.
7. SVG snapshots show no clipped labels or overlapping nodes in the golden corpus.
8. Diagram meaning remains understandable in grayscale.
9. Text and essential strokes meet the selected WCAG AA contrast targets.
10. The manifest records all versions, input hashes and renderer options needed to reproduce the build.
11. Renderer failures and timeouts return documented non-zero exit codes and do not report success based on partial output files.
12. The AI evaluation corpus reaches an agreed schema-validity threshold and does not introduce raw styling fields.
13. One documented command builds every example from source.

The exact AI quality threshold and render-performance budget MUST be set after Stage 0 establishes a baseline. They must be specified before v0.1 is declared complete.

---

## 23. Quality and test strategy

### 23.1. Test categories

- JSON Schema positive and negative fixtures,
- semantic-validator unit tests,
- projection tests for every family,
- deterministic D2 golden tests,
- renderer integration tests,
- normalized SVG snapshots,
- offline-build tests,
- accessibility checks,
- CLI exit-code tests,
- security tests for unsafe YAML, path traversal and remote assets,
- AI evaluations separated from deterministic core tests.

### 23.2. Golden corpus

The MVP corpus SHOULD contain at least:

- five infrastructure views,
- five integration-flow views,
- five sequence views,
- small, medium and boundary-case diagrams,
- nested boundaries,
- missing optional icons,
- long labels and Unicode labels,
- intentionally invalid models for diagnostic tests.

Human visual review remains part of release approval until reliable automated layout-quality metrics are established.

---

## 24. Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Layout changes between D2 versions | noisy diffs | pin D2, normalize snapshots, explicit upgrades |
| TALA licensing or installation | CI and distribution constraints | ELK baseline; TALA optional |
| Model becomes a full architecture repository | excessive scope | narrow schemas, explicit non-goals, family IRs |
| AI invents architecture facts | misleading diagrams | uncertainty reporting, source-conformance mode, validation |
| Overloaded semantic vocabulary | inconsistent diagrams | orthogonal fields and documented enums |
| Remote icons break offline builds | non-reproducible output | local approved assets, remote resources disabled |
| D2 classes are overridden | inconsistent styling | generated D2 only, linting, no style fields in source schemas |
| Automatic review changes semantics | architecture corruption | immutable System Model during layout improvement, visible diff |
| Sequence requirements outgrow static relations | incompatible model | explicit ordered scenarios and Sequence IR |
| Large diagrams remain unreadable | poor usability | view filtering, warnings, future aggregation |

---

## 25. Open decisions

The following decisions must be resolved during Stage 0 or Stage 1:

1. Exact pinned D2 version and upgrade cadence.
2. Python packaging and dependency-management tool.
3. Whether generated SVG files are committed or produced only in CI.
4. Target documentation renderers and browsers.
5. SVG normalization strategy for snapshot tests.
6. Initial generic icon set and its license.
7. Exact policy for displaying technology names and protocols.
8. AI evaluation threshold and representative prompt corpus.
9. Performance budget for small and medium diagrams.
10. Whether v0.1 needs a dark theme or only the light baseline.

---

## 26. First milestone

The first practical milestone is a contract-first vertical slice:

```text
FlowFrame v0.1-alpha.1
```

It MUST accept:

```text
examples/payments/system-model.yaml
examples/payments/infrastructure-view.yaml
```

and produce with one command:

```text
build/payments/infrastructure/
├── diagram.d2
├── diagram.svg
└── manifest.json
```

The milestone validates the central concept only when the same model can subsequently generate at least one flow or sequence view without duplicating the system inventory.
