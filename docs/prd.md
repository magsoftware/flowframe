# FlowFrame — Product Requirements

## 1. Document status

| Field | Value |
|---|---|
| Product | FlowFrame |
| Target release | v0.1 MVP |
| Status | Draft for implementation |
| Primary output | SVG |
| Baseline layout engine | ELK |
| Post-MVP layout engine | TALA |

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
- an optional project-wide `flowframe.yaml` configuration with a built-in default when absent,
- a reusable System Model,
- a View Specification selecting and presenting part of that model,
- three initial view types:
  - `architecture/infrastructure`,
  - `flow/integration-flow`,
  - `sequence/sequence`,
- deterministic generation of D2 source,
- SVG rendering through a pinned D2 CLI,
- ELK as the portable baseline layout engine,
- a centralized light theme and one project-wide custom Theme/Brand Pack,
- a small, documented semantic vocabulary,
- semantic validation in addition to schema validation,
- local and CI core execution without network access (optional AI adapters may use a network),
- read-only `review` modes `syntax`, `semantic` and `policy`, plus optional AI source-conformance review,
- explicit multi-view project builds,
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
- model migration tooling,
- whole-project validation without rendering (`validate --all`, `review --all`),
- element-count guideline warnings (section 14),
- boundary-scoped selection such as `select.within`,
- flow source/sink role annotations,
- glyph-coverage validation for fonts,
- machine-readable SPDX license inventories,
- Python API entry points for `render` and `review`.

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

AI MUST NOT put raw colors, fonts, line weights, arbitrary shapes or manual coordinates in System Models or Views. Explicitly authorized project-wide branding edits follow section 5.

AI MAY describe:

- component meaning and technology,
- boundaries and containment,
- relations and interaction properties,
- scenario messages,
- optional semantic descriptions and view purpose,
- diagram intent and audience,
- portable layout preferences exposed by FlowFrame.

---

## 5. Source of truth and artifacts

For v0.1, the source of truth is:

```text
[flowframe.yaml + selected Theme/Brand Pack] + system-model.yaml + view.yaml
```

`flowframe.yaml` is optional. When it is absent, FlowFrame MUST use the versioned built-in default configuration and light theme. When present, it applies globally to every diagram built within that project root.

The following files are generated artifacts:

```text
diagram.d2
diagram.svg
manifest.json
```

Generated D2 MUST contain a header stating that it must not be edited manually. Rebuilding a diagram MAY overwrite generated artifacts.

AI MUST produce or update only structured source files. The deterministic compiler is solely responsible for D2 generation and visual styling.

Project maintainers own `flowframe.yaml` and custom Theme/Brand Packs. AI MUST NOT create or modify raw theme-token values unless the user explicitly requests a project-wide branding change.

Hand-authored D2 MAY be accepted in a future compatibility mode, but that mode will not receive the same semantic and styling guarantees.

---

## 6. Technology decisions

### 6.1. Diagram language: D2

D2 is the rendering target because it provides:

- readable text syntax,
- SVG generation,
- reusable classes and variables,
- reusable diagram primitives (FlowFrame emits self-contained D2; source imports are post-MVP),
- icons and containers,
- sequence diagrams,
- multiple layout engines,
- suitable primitives for software architecture diagrams.

D2 is an implementation dependency, not the public FlowFrame authoring format. FlowFrame schemas MUST remain independent enough to allow another renderer in the future.

### 6.2. Layout engines

**ELK is the baseline and default layout engine.** It is bundled with D2, works well for hierarchical diagrams and does not require a separate commercial license.

**TALA is optional.** It may produce better results for non-hierarchical architecture diagrams, but it is separately installed, closed-source, requires a commercial license for commercial use and can produce different layouts after small input changes.

The framework MUST NOT silently switch layout engines based on a subjective assessment of visual quality.

v0.1 supports only `elk`: `render.layoutEngine` in Project Configuration defaults to `elk`; `build --layout` and `render --layout` override it explicitly (precedence: option > configuration > `elk`). An unsupported engine on the command line is a usage error (2); an unsupported engine in `flowframe.yaml` is a validation error (1); missing installed dependencies return code 3. There is no fallback option in v0.1 ([ADR-0003](adrs/0003-elk-baseline-optional-tala.md)). Any post-MVP fallback requires a separate explicit contract and records both requested and effective engines.

The post-MVP comparison command MAY render the same view with two or more explicitly selected installed engines; it never selects all installed engines implicitly.

### 6.3. MVP implementation

The recommended MVP implementation is a small Python 3.12+ CLI ([ADR-0002](adrs/0002-python-cli-and-d2-process-boundary.md)) that:

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
   Project Configuration + System Model + View
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
       SVG decoration compositor
                 │
                 ▼
       normalize / verify / publish
                 │
                 ▼
      diagram.svg + manifest.json
                 │
                 ▼
        lint / snapshot / review
```

### 7.1. Required separation of responsibilities

- **System Model** describes reusable facts about the system.
- **Project Configuration** selects one global Theme/Brand Pack and decoration policy for every diagram in the project.
- **View Specification** describes what to show, for whom and the textual content of optional presentation elements.
- **Projection** selects, aggregates and normalizes information for a diagram family.
- **Family IR** captures information unique to architecture, flow or sequence diagrams.
- **D2 generator** maps the normalized IR to framework-owned classes and templates.
- **Decoration compositor** adds title, subtitle, source footer, project logo, legend and SVG accessibility metadata using the global presentation policy ([ADR-0005](adrs/0005-presentation-composition-outside-d2.md)).
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
schemaVersion: flowframe-model/v1
system:
  id: payments-platform
  label: Payments Platform

boundaries:
  - id: production
    kind: environment
    label: Production
    parentId: payments-platform

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
- IDs MUST match `^[a-z][a-z0-9]*(-[a-z0-9]+)*$` and contain 1–64 ASCII characters.
- System, boundaries, elements, relations, scenarios and all scenario steps share one model-wide ID namespace. Every object MUST have an explicit ID.
- A View ID has its own project-wide namespace, enforced across the `views` registry and any View passed on the command line. Tags, View IDs, theme IDs and brand asset IDs use the same grammar as model IDs; tag lists contain no duplicates.
- A theme `version` is a string matching `^[0-9A-Za-z.+-]{1,32}$`.
- Source IDs that coincide with D2 keywords remain valid: the generator MUST prefix and escape them using a collision-free internal namespace.
- Labels are human-readable and MAY contain Unicode.

### 8.4. Common metadata and containment

Elements, boundaries, relations, scenarios and the system MAY carry a plain-text `description`. Elements MAY declare `status: active | deprecated`, default `active`. A View MAY carry `purpose`. There is no generic priority or emphasis field in v0.1. AI assumptions are delivered in a separate human-readable `assumptions.md` report, never as undeclared YAML fields ([ADR-0025](adrs/0025-ai-adapter-boundary.md)).

All human-readable source text follows one rule: it is NFC-normalized before length checks, contains no control characters (Unicode category Cc, including line breaks), and a required value is non-empty after trimming whitespace. Limits count Unicode code points after NFC normalization:

| Field | Maximum |
|---|---:|
| `label` of system, boundary, element, relation, scenario and scenario step | 120 |
| `technology` of element or relation | 80 |
| `protocol` of relation or message | 120 |
| `payload` | 500 |
| `description`, View `purpose` | 2,000 |
| theme asset `alt` | 200 |

The `[deprecated]` marker and composed label parts (section 13.10) do not count toward these limits.

`parentId` on an element or boundary references only a boundary or `system.id`; elements cannot be parents. Missing `parentId` means a root-level object outside the explicit system boundary. Boundaries may nest freely by kind, with one exception: a `subsystem` boundary MUST have `system.id` in its ancestor chain. Cycles are invalid. `actor` and `external-system` MUST NOT have the system in their ancestor chain. Membership is explicit, not inferred from kind: an element or boundary intended to belong directly to the system MUST declare `parentId: <system.id>`; a nested member reaches the system through its ancestors. `display.systemBoundary` controls visibility, not membership. If it is true but no selected element belongs to the system, emit an FFV warning explaining the missing ancestry and omit the empty system boundary. Validation cannot infer an author's unstated intent.

A model without intermediate boundaries still declares membership explicitly:

```yaml
schemaVersion: flowframe-model/v1
system:
  id: example-system
  label: Example system
elements:
  - id: client
    kind: external-system
    label: External client
  - id: api
    kind: service
    label: API
    parentId: example-system
```

Here `client` is outside and `api` is inside the system. Model collection fields (`boundaries`, `elements`, `relations`, `scenarios`) default to empty lists when omitted; view/scenario validation still rejects empty required results.

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
payload
```

`payload` is optional plain text (maximum 500 code points) naming transferred data or an event; it does not create another model element.

`id`, `source`, `target`, `semantic` and `interaction` are required for every relation; endpoints MUST reference elements, never boundaries. `source` and `target` are required for every relation, including an `undirected` relation. They identify the two endpoints and provide stable serialization and traceability. For `directed` and `bidirectional` relations, their order defines the normal forward order; reverse flow is represented by swapping them, not by another enum value. For `undirected`, their order has no flow meaning. `directionality` MAY be `directed`, `bidirectional` or `undirected`, defaults to `directed` and controls the directional interpretation and arrowhead rendering without changing endpoint requirements.

`encrypted` is a boolean describing transport encryption for that relation. It does not assert broader end-to-end or at-rest encryption.

`internet` is not a relation type. Model the Internet as a root-level `network` boundary when grouping nodes, or describe the route in a relation label. A boundary cannot be a traffic endpoint; examples requiring an external endpoint MUST model the actual external client/system instead. New `client-application`, `region` and `account` kinds are post-MVP; use `external-system` for an external client and tags/descriptions for deployment classifications.

`trust` is not a generic connection type. Trust boundaries and authentication/authorization relations MUST be modeled explicitly.

### 9.3.1. Allowed relation combinations

| Semantic | Allowed interaction | Allowed directionality |
|---|---|---|
| `dependency` | `not-applicable` | directed, bidirectional, undirected |
| `request`, `control`, `authentication`, `authorization` | synchronous, asynchronous | directed, bidirectional |
| `event` | asynchronous, synchronous (warning for synchronous) | directed, bidirectional |
| `data-access` | synchronous, asynchronous | directed, bidirectional |
| `data-flow`, `replication` | synchronous, asynchronous, not-applicable | directed, bidirectional |

All unlisted combinations are semantic errors. `not-applicable` is permitted for descriptive transfers where timing is not asserted; it is not a synonym for asynchronous. Missing required properties are errors, not inferred facts. `encrypted: true/false`, when displayed, adds “transport encrypted” / “transport unencrypted”; absence means unknown and adds no assertion.

### 9.4. Scenario step kinds

The v0.1 scenario vocabulary is:

| Kind | Required fields | Meaning |
|---|---|---|
| `message` | `id`, `from`, `to`, `label` | An ordered interaction between two participants. A self-message uses the same ID in `from` and `to`; it is not a separate kind. |
| `note` | `id`, `participant`, `label` | An explanatory note attached to one participant at that point in the scenario. `participant` MUST reference an element ID; boundary IDs are invalid. |

Message `from` and `to` MUST reference elements. Optional message fields are `protocol` (plain text, maximum 120 code points) and `relationId`. A referenced relation MUST connect those participants: directed relations must match forward order, bidirectional relations permit either order, and undirected relations cannot back a message. An explicit protocol must equal the referenced relation protocol when both exist; otherwise the relation protocol is inherited. No other relation property is silently copied into scenario timing, order or appearance. Notes have no protocol or relation reference. Every message has one solid line and a single arrowhead from `from` to `to`; in the sequence family this expresses an ordered message, not synchronous timing. A response is a separate reverse message with an explicit label, not an inferred return arrow. `relationId` does not change its style, even when the referenced relation is bidirectional or asynchronous. If enabled, the sequence legend explains only the message arrow, note and step numbering actually shown; it never invents timing semantics. Message and note label composition, including step numbering, follows section 13.10.

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
- optional selection criteria for architecture/flow, or a required `scenarioId` for sequence,
- optional display options and layout settings, with the defaults below.

Engine-specific authoring overrides are post-MVP.

A view MAY also declare presentation content such as a title, subtitle and source attribution. It MUST NOT select a theme, logo, font, color or decoration position.

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
| `low` | Names and selected boundaries/relations; protocol and technology details hidden. |
| `medium` | Boundaries and relation labels visible; selected technology and protocol details shown when useful. |
| `high` | All supported metadata relevant to the selected family is shown unless explicitly disabled. |

The detail preset MUST NOT add or remove selected elements. Explicit `display` values override preset defaults. Audience is metadata used by review rules and AI guidance; in v0.1 it MUST NOT change selection, generated D2 or rendered SVG. Exact defaults are defined in section 10.8.

### 10.4. Architecture view example

```yaml
schemaVersion: flowframe-view/v1
id: infrastructure-overview
family: architecture
subtype: infrastructure
audience: architect
detail: medium

presentation:
  title: Payments Platform
  subtitle: Production infrastructure
  source: Architecture Team, 2026-09-25

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
schemaVersion: flowframe-view/v1
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

A sequence view references exactly one `scenarioId` in v0.1 and MUST NOT declare `select`, `exclude`, `display.boundaries` or `display.systemBoundary`. Every referenced participant is included. Default participant order is first occurrence in scenario steps (message `from` before `to`; note `participant` at its step). Optional View `participants: [element-id, ...]` MUST be an exact, duplicate-free permutation of those participants. Boundaries are not rendered. Combining scenarios and filtering scenario steps are post-MVP.

### 10.6. Presentation metadata

The optional `presentation` object provides diagram-specific text while all styling and placement remain global:

| Field | Limit | Meaning |
|---|---:|---|
| `title` | 120 characters | Primary diagram title. |
| `subtitle` | 200 characters | Optional secondary context. |
| `source` | 500 characters | Source attribution inserted into the configured footer template. |

Values are always interpreted as plain text and escaped for each output context, including strings such as `a < b` or `<svg>`. Limits count Unicode code points after NFC normalization. Control characters (Unicode category Cc) are invalid; automatic wrapping inserts layout breaks without changing source content. Missing values omit the corresponding text element. A `subtitle` without a `title` is an FFV validation error. The project logo is controlled exclusively by Project Configuration and cannot be replaced or disabled per view in v0.1.

### 10.7. Selection semantics

This algorithm applies only to architecture and flow:

1. Treat absent or empty element filter lists as unpopulated. Build seeds from `select.ids`, `select.tags` and `select.kinds`: OR within a field, AND between populated fields. If none is populated, seed all elements. IDs reference elements only, and `select.kinds`/`exclude.kinds` accept element kinds only (a boundary kind is a schema error). Selecting by boundary membership is post-MVP.
2. Resolve `exclude.ids/tags/kinds` with OR between and within fields, before traversal. Excluded elements are barriers and are removed from seeds. Warn if an explicitly selected ID is excluded.
3. Apply the family relation-semantic defaults in section 10.8 (the table row `select.relations.semantics`), then filter candidate relations by `select.relations.ids/semantics/interactions/tags` (OR within, AND between populated fields). Remove matches of `exclude.relations` with the same fields (OR between and within fields); `exclude.relations` applies whether or not `select.relations` is present. Empty exclusion lists exclude nothing. Missing IDs are errors.
4. `includeRelated` defaults to false. If true, traverse candidate relations breadth-first in both directions, independently of arrowheads, without entering excluded nodes. `relatedDepth` defaults to 1 and is bounded to 1–10; supplying it when traversal is disabled is invalid.
5. Include candidate relations only when both endpoints are selected.
6. Add containment ancestors of selected elements. Display only requested boundary kinds and, when enabled, the system boundary; promote descendants of hidden boundaries to their nearest visible ancestor or the view root. No empty boundary is created by this operation.

An empty result is an FFV validation error. Disconnected selected components are allowed and produce a warning. Containment-depth filtering and aggregation are post-MVP.

### 10.8. Defaults and family field catalog

`schemaVersion`, `id`, `family`, `subtype`, `audience` and `detail` are required. `select`, `exclude`, `display`, `layout`, `presentation` and `purpose` are optional where permitted by family.

| Field | Architecture | Integration flow | Sequence |
|---|---|---|---|
| `layout.profile` | hierarchical (compact also allowed) | hierarchical (compact also allowed) | sequence only |
| `layout.direction` | right | right | forbidden; order is defined by scenario/participants |
| `display.boundaries` | all six boundary kinds | [] | forbidden |
| `display.systemBoundary` | true | false | forbidden |
| `display.relationLabels` | true; optional user labels only | true; optional user labels only | forbidden; message labels always visible |
| `display.protocols` | low: false; medium/high: true | low: false; medium/high: true | low: false; medium/high: true |
| `display.technologies` | low: false; medium/high: true | low/medium: false; high: true | low/medium: false; high: true |
| `display.payloads` | false | true | forbidden |
| `display.encryption` | high only | high only | forbidden |
| `display.stepNumbers` | forbidden | forbidden | true |
| `display.legend` | true | true | false |
| `select.relations.semantics` (when absent or empty) | all semantics | all except `dependency` | forbidden |

The table is the complete v0.1 display catalog. All defaults are applied by one resolver from a single data source ([ADR-0019](adrs/0019-default-value-resolution.md)). Explicit booleans override detail presets. Empty `display.boundaries` hides all boundary kinds. Detail does not filter relations or elements. Unknown family fields are errors.

Integration-flow uses the same selection algorithm, defaults its relation semantics to all except `dependency` when `select.relations.semantics` is absent or empty, and requires directed or bidirectional selected relations. An explicit dependency filter is permitted only for directed/bidirectional dependencies. If an undirected dependency survives the relation filters and has both endpoints selected, validation fails with an FFV diagnostic at the filter/relation and a hint to exclude that relation or use an architecture view. It MUST NOT be silently dropped or downgraded to a warning. An undirected dependency elsewhere in the model does not invalidate the flow view. Its projection omits infrastructure boundaries by default, shows payload labels by default and distinguishes stores (`database/cache/storage`) from processing and messaging nodes by their semantic shape and icon (section 13.9), not by extra label text. Source/sink role annotations are post-MVP; no business facts are inferred.

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

- selected nodes with semantic kinds,
- directed or bidirectional flows (not an invented execution order),
- relation payload labels where present,
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

The capability table documents engine differences; TALA and `layout.engineOptions` are post-MVP. v0.1 rejects engine-specific View options. `compact` is a framework-owned spacing preset, not a promise of exact placement; [ADR-0003](adrs/0003-elk-baseline-optional-tala.md) records the D2 mapping for both portable profiles.

Manual coordinates are outside the MVP contract.

---

## 13. Design system

### 13.1. Global customization policy

All visual properties MUST be owned by framework libraries, one project-wide Theme/Brand Pack and the D2 generator. System Models and View Specifications MUST NOT contain raw D2 style declarations, RGB/HEX values, font names, logo paths or per-diagram theme selection.

A project uses exactly one active theme for a build. Per-view theme or brand overrides are invalid in v0.1. Raw visual values are permitted only inside a validated `theme.yaml` belonging to the selected Theme/Brand Pack.

An element class MAY be derived from:

- semantic kind (shape, token mapping and generic icon; sections 13.8–13.9),
- declared `status: deprecated`.

Free-text `technology` never selects a class or icon.

### 13.2. Project Configuration

An optional project-root `flowframe.yaml` applies to every diagram in that project. Its v0.1 contract includes the selected theme and global decoration policy:

```yaml
schemaVersion: flowframe-config/v1
theme: company-light
render:
  layoutEngine: elk
views: [infrastructure-view.yaml, integration-flow-view.yaml, sequence-view.yaml]

branding:
  logo:
    asset: company-logo
    position: top-right

  title:
    position: top-center

  footer:
    enabled: true
    template: "Source: {source}"
    position: bottom-left
```

Allowed positions are:

- logo: `top-left` or `top-right`,
- title: `top-left` or `top-center`,
- footer: `bottom-left`, `bottom-center` or `bottom-right`.

Slot-occupancy constraints from section 13.6 are part of Project Configuration validation and are evaluated after built-in defaults are applied. In particular, title and logo cannot both occupy `top-left`; because the default title position is `top-left`, a configuration that places the logo at `top-left` MUST also set `branding.title.position: top-center`. The diagnostic states when a conflicting position came from a default.

The footer template MAY contain static text and zero or one `{source}` placeholder. A template without `{source}` is rendered as a static footer whenever the footer is enabled. A template containing `{source}` is omitted when the view has no `presentation.source`.

Template parsing uses these deterministic rules:

- `{source}` inserts the source value as escaped plain text,
- `{{` produces a literal `{`,
- `}}` produces a literal `}`,
- any unmatched single brace, unknown placeholder or second `{source}` occurrence is a validation error,
- braces contained in the source value are data and are not parsed again.

The CLI MUST accept an explicit `--config` path. Without it, FlowFrame searches upward from the System Model directory for the nearest `flowframe.yaml`, stopping after checking the nearest VCS root (`.git` file or directory). Outside VCS it checks only the model directory; parent discovery requires explicit `--config`. If none exists, it uses the versioned built-in configuration.

Optional `views` is an explicit list of project-root-relative View file paths used by `build --all`; it defaults to `[]`. Paths must stay inside the project root and resolve to distinct files with distinct View IDs. All registered views use the model supplied to the command. There are no globs in v0.1.

The project root is the directory containing the resolved `flowframe.yaml`. A sibling `model/` and `views/` structure therefore needs a `flowframe.yaml` in their common parent (found by discovery or passed with `--config`). Without a config, the model directory is the root; a View outside it fails with an FFC validation diagnostic (exit 1) suggesting that common-parent configuration. A symlink or theme asset that escapes the project root is a policy error (exit 5). Project theme IDs resolve from `<project-root>/themes/<theme-id>/`. If no project configuration exists, the System Model directory is the effective project root and only built-in themes are considered. Built-in themes are immutable package resources shipped inside the installed FlowFrame distribution; they do not live in the project `themes/` directory. Built-in theme IDs are reserved: a directory `<project-root>/themes/<built-in-id>/` is an error whenever it exists, even if the configuration selects another theme. Other IDs resolve only from the project theme directory. The theme's declared `id` MUST equal its directory name. The resolved configuration path or built-in ID and the resolved theme origin MUST be recorded in the manifest.

#### Built-in configuration `flowframe-default`, version 1

`schemaVersion` is required in a file; all other fields default recursively to the following values. An absent logo is disabled; a present logo requires an asset ID and defaults to `top-right`.

```yaml
schemaVersion: flowframe-config/v1
theme: flowframe-light
render:
  layoutEngine: elk
views: []
branding:
  title:
    position: top-left
  footer:
    enabled: true
    template: "Source: {source}"
    position: bottom-left
```

An omitted title or subtitle is not synthesized. No source means no default footer. Built-in configuration identity and version are recorded even when no file exists.

### 13.3. Theme/Brand Pack

A Theme/Brand Pack is a versioned, project-wide package:

```text
themes/company-light/
├── theme.yaml
├── assets/
│   ├── logo.svg
│   └── fonts/
└── LICENSES.md
```

`theme.yaml` MUST declare:

- schema version, theme ID and theme version,
- design-token values,
- semantic mappings from element kinds and relation semantics to tokens,
- local font asset references when custom fonts are used,
- named brand assets with local paths, media types, accessible alternative text and bounded display dimensions,
- decoration tokens for title, subtitle, footer and logo spacing.

Example:

```yaml
schemaVersion: flowframe-theme/v1
id: company-light
version: "1"

tokens:
  background: "#FFFFFF"
  surface: "#F7F9FC"
  surface-muted: "#EEF2F7"
  border: "#54606E"
  text-primary: "#17202A"
  text-secondary: "#52606D"
  title-text: "#17202A"
  subtitle-text: "#52606D"
  footer-text: "#52606D"
  decoration-gap: 16
  decoration-padding-x: 24
  decoration-text-max-width: 720
  compute: "#DCEEFF"
  database: "#E8DCF8"
  messaging: "#FFF1CC"
  storage: "#DFF3E4"
  security: "#FFE4C4"
  external: "#ECEFF1"
  network: "#DDEBF7"
  line-request: "#245B8A"
  line-event: "#8A5A00"
  line-data: "#5B3F8C"
  line-dependency: "#59636E"
  line-control: "#8A2F2F"

mappings:
  elements:
    service: compute
    worker: compute
    database: database
    queue: messaging
  relations:
    request: line-request
    event: line-event
    data-access: line-data

typography:
  fontSet: flowframe-default

licenses: LICENSES.md

assets:
  company-logo:
    path: assets/logo.svg
    mediaType: image/svg+xml
    alt: Company logo
    maxWidth: 120
    maxHeight: 48
```

The project configuration references brand assets by ID, never by arbitrary file path. Theme resolution and reserved IDs follow section 13.2. Remote themes, fonts and logos are not supported in v0.1.

Semantic mappings MAY override a subset of framework defaults. Any unmapped kind or relation semantic falls back to versioned framework mappings, independent of the selected built-in theme. All fallback mappings use required tokens; an unknown token reference is a validation error.

Raw RGB/HEX values are allowed only as token values in `theme.yaml`. Component shapes and relation semantics remain framework-owned and cannot be replaced by arbitrary D2 code in a Theme/Brand Pack.

#### Theme field types and complete default mappings

All color tokens listed below are required opaque `#RRGGBB` sRGB strings. `decoration-gap` and `decoration-padding-x` are numeric CSS pixels, default 16 and 24, bounded to 0–128. `decoration-text-max-width` is numeric CSS pixels, default 720, bounded to 240–1440. These three numeric fields are not colors.

`typography` defaults to `{fontSet: flowframe-default}`, an immutable `resources/fonts/flowframe-default/` font set with pinned bytes and license. Its font family and the scripts it covers are documented by [ADR-0007](adrs/0007-svg-text-and-fonts.md). Glyph coverage is not validated in v0.1: text outside the documented coverage renders with missing-glyph boxes. When supplied, `typography` MUST contain exactly one of `fontSet` or `fonts`; both together and an empty object are errors. The custom variant is `typography.fonts`, with all four local faces required and no host fallback:

```yaml
typography:
  fonts:
    regular: assets/fonts/regular.ttf
    bold: assets/fonts/bold.ttf
    italic: assets/fonts/italic.ttf
    semibold: assets/fonts/semibold.ttf
```

v0.1 custom fonts use TTF; [ADR-0007](adrs/0007-svg-text-and-fonts.md) verifies D2 support and selects the output-wide text policy. Font sizes are framework-owned in v0.1. `licenses` is required and references a local UTF-8 `LICENSES.md` listing path, license name and source for every font, logo and other asset; it is included in the aggregate hash. Its content is reviewed manually; machine-readable SPDX inventories are post-MVP ([ADR-0024](adrs/0024-license-and-provenance-inventory.md)).

| Mapping category | Source kinds/semantics | Required token |
|---|---|---|
| elements | actor, external-system | external |
| elements | web-application, service, worker, gateway | compute |
| elements | database, cache | database |
| elements | queue | messaging |
| elements | storage | storage |
| elements | identity-provider, secret-store, security-control | security |
| relations | request | line-request |
| relations | event | line-event |
| relations | data-access, data-flow, replication | line-data |
| relations | dependency | line-dependency |
| relations | control, authentication, authorization | line-control |
| boundaries | network | network |
| boundaries | trust-zone | security |
| boundaries | subsystem, environment, cluster, namespace | surface-muted |

`mappings.elements`, `mappings.relations` and `mappings.boundaries` may override these token references only. The system boundary uses `surface` and `border`. Theme mappings never change shapes, line patterns, arrowheads or meaning. How each token is applied is defined in section 13.9.

### 13.4. Design tokens

The resolved theme MUST contain the following tokens. In the source `theme.yaml`, color tokens are required; the three numeric decoration tokens MAY be omitted and are filled with the defaults from section 13.3 before validation of the resolved theme and layout:

```text
background
surface
surface-muted
border

text-primary
text-secondary
title-text
subtitle-text
footer-text
decoration-gap
decoration-padding-x
decoration-text-max-width

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

### 13.5. Relation appearance

Relations MUST be distinguishable using more than color. The mapping SHOULD use a combination of:

- stroke pattern,
- arrowhead,
- thickness,
- label prefix or text,
- color.

For architecture and flow, the exact non-color mapping belongs to the versioned framework, not the theme: synchronous relations use solid strokes, asynchronous relations dashed strokes, and not-applicable relations dotted strokes; directionality alone controls arrowheads (forward, both, none). Labels include the semantic name even when `display.relationLabels: false` hides the optional user-supplied relation `label`; protocol, technology, payload and encryption text are controlled independently by their own display flags and composed as defined in section 13.10. A legend explains the styles actually present. Encryption is expressed as text, never color alone. Themes control the mapped colors; line weights and arrowhead geometry are framework-owned in v0.1. Sequence message appearance follows section 9.4, not this interaction-to-pattern mapping.

An element with `status: deprecated` always adds the literal text marker `[deprecated]` to its displayed label and accessible description in every family. It may also use a framework-owned visual distinction, but neither color nor a legend replaces the text marker.

### 13.6. Diagram decorations

The decoration generator MAY add:

- the view title and subtitle,
- the source footer produced from the global template,
- the project logo selected in `flowframe.yaml`.

Decorations MUST use global theme tokens/styles and configured positions. They are presentation objects, not System Model elements and MUST NOT participate in semantic selection or the 7–9 element guideline. When enabled, a generated legend describes the relation patterns/arrowheads used in the diagram; entries follow the vocabulary order and occupy a measured bottom band above the footer. An empty legend occupies no space. Legend entries, sample geometry and the sequence variant are defined in [ADR-0015](adrs/0015-legend-rendering.md); the D2 generator and the legend share one line-style definition.

Decoration space is reserved by a deterministic SVG composition step rather than by ELK or TALA:

1. Render and measure the semantic diagram body.
2. Wrap title/subtitle/footer/legend text greedily at whitespace using pinned font metrics up to `decoration-text-max-width`; split overlong runs at Unicode grapheme boundaries, preserve all text and reject an unfit single grapheme with an actionable diagnostic. No truncation is permitted.
3. Render wrapped text and the fitted logo as independently measurable SVG fragments. Calculate a top band from the maximum title-block/logo height, and separate stacked bottom bands for legend and footer. Add `decoration-gap` on both sides of each occupied band.
4. Let L, C, R be measured occupied top-slot widths, G the horizontal gap, P the horizontal padding and F the footer/legend maximum width. For a centered title, the minimum inner width is `C + 2 × (max(L,R) + G)` when a side slot is occupied, otherwise C. Without a centered title it is the sum of occupied side widths plus G when both exist. Final width is `max(bodyWidth, topMinimum, F) + 2 × P`.
5. Expand the final canvas and `viewBox`, then translate the semantic body below the top band. Center the body and center-slot title on the final width; align side slots inside P. Top blocks are vertically centered within the same band. D2 padding belongs to body bounds and MUST NOT be subtracted a second time.

A top slot may contain only one primary decoration; configuring both logo and title for `top-left` is a validation error. The subtitle is part of the title block. The composition step MUST run before final SVG normalization and snapshot comparison. It MUST NOT ask the layout engine to position decorations.

A logo MUST be a local approved SVG asset from the active Theme/Brand Pack. It MUST be embedded or bundled into the output so that SVG rendering remains offline and portable. Its accessible alternative text comes from theme asset metadata.

Asset-level `maxWidth` and `maxHeight` are positive CSS-pixel bounds for the rendered logo box. The composer preserves the intrinsic aspect ratio and scales the logo up or down to the largest size that fits within that box ("contain"). These requested bounds MUST remain below implementation-wide security limits for dimensions, source bytes, element count and nesting depth; a Theme/Brand Pack cannot raise the global limits.

### 13.7. Accessibility

The v0.1 theme MUST:

- meet at least 4.5:1 contrast for text against its actual background and 3:1 for essential non-text boundaries/strokes against adjacent backgrounds (logos are exempt from these contrast thresholds),
- remain understandable in grayscale,
- avoid color as the only semantic signal,
- use readable default font sizes,
- provide visible relation labels where required,
- include a deterministic top-level SVG `<desc>` summarizing selected element IDs/labels and relations, or ordered scenario steps, plus descriptions for visible objects; this is not a separate required file,
- expose the diagram title and logo alternative text in the final SVG accessibility metadata, added by the compositor independently of renderer support.

The summary template, per-object metadata and ARIA attributes are defined in [ADR-0014](adrs/0014-svg-accessibility-metadata.md) and documented in `rules/accessibility-rules.md`. View `purpose` and object `description` values feed the summary and per-object descriptions. These requirements apply to `build` output; `render` body SVG carries no summary.

Contrast pairs checked deterministically are listed in section 13.9.

Failed deterministic contrast checks are FFT validation errors; consumer-dependent appearance still requires visual review. These numerical checks support accessibility but do not alone certify full WCAG conformance.

### 13.8. Icons

```text
icons/
├── generic/
├── azure/       # post-MVP
├── aws/         # post-MVP
├── kubernetes/  # post-MVP
└── security/    # post-MVP vendor/specialist pack
```

Rules:

- generic local SVG icons are the MVP baseline,
- remote icon URLs are disabled by default,
- rendering MUST work offline,
- each icon pack MUST include license and attribution metadata,
- vendor logos MUST follow the vendor's trademark rules,
- technology without an approved icon falls back to its semantic shape,
- icon absence MUST NOT change component semantics.

The MVP generic pack has a versioned `index.json` mapping each element kind to a local icon asset or explicit `null`, with SHA-256 and license references. Mapping is by semantic kind, never guessed from free-text `technology`. No source `icon` or `technologyId` field is introduced. Icons are sanitized with the `flowframe-svg-logo/v1` profile when the FlowFrame package is built; at runtime FlowFrame verifies each used icon against its index hash ([ADR-0013](adrs/0013-generic-icon-pack.md)). The compiler embeds the verified icons as bounded SVG data URIs in generated D2, once per unique asset identity (SHA-256), using reusable declarations/classes rather than repeating the URI per node. P0 must prove this reuse works offline with the pinned D2. An explicit `null` means intentionally no icon and uses the semantic shape without a warning. A missing kind entry, a declared file that is absent, or a hash mismatch is an invalid pack and an error, not a silent fallback.

### 13.9. Shapes and token application

Element kinds map to framework-owned D2 shapes so that kinds stay distinguishable without color or icons:

| Kind | Shape | Additional non-color cue |
|---|---|---|
| actor | person | — |
| external-system | rectangle | dashed border |
| web-application | page | — |
| service | rectangle | — |
| worker | step | — |
| gateway | hexagon | — |
| database | cylinder | — |
| cache | stored_data | — |
| queue | queue | — |
| storage | package | — |
| identity-provider | oval | — |
| secret-store | diamond | — |
| security-control | parallelogram | — |

Boundaries are rectangular containers; `trust-zone` uses a dashed border, other kinds a solid border. Tokens are applied as follows:

| Object | Fill | Stroke | Text |
|---|---|---|---|
| Canvas | `background` | — | — |
| Element | mapped element token | `border` | `text-primary` |
| Boundary | mapped boundary token | `border` | `text-primary` |
| System boundary | `surface` | `border` | `text-primary` |
| Relation / sequence message | — | mapped relation token (messages: `line-request`) | `text-primary` |
| Sequence note | `surface-muted` | `border` | `text-primary` |
| Title / subtitle / footer | — | — | `title-text` / `subtitle-text` / `footer-text` |
| Legend | — | relation tokens for samples | `text-secondary` |

The theme validator checks these pairs from the resolved mappings (FFT errors):

- text at 4.5:1: `text-primary` against every element and boundary fill token, `surface`, `surface-muted` and `background`; `title-text`, `subtitle-text`, `footer-text` and `text-secondary` against `background`,
- non-text at 3:1: every `line-*` token and `border` against `background`, `surface`, `surface-muted`, `network` and `security`.

Details and rationale: [ADR-0012](adrs/0012-shapes-and-token-application.md).

### 13.10. Label composition

All labels are plain text composed by one framework module ([ADR-0011](adrs/0011-label-composition.md)). Hidden or absent parts are omitted together with their separator; the inline separator is ` · `.

| Object | Line 1 | Line 2 |
|---|---|---|
| Element or participant | `<label>` plus ` [deprecated]` when deprecated | `[<technology>]` when `display.technologies` |
| Boundary | `<label> (<kind>)`; system: `<label> (system)` | — |
| Relation (architecture, flow) | `<semantic>`, plus `: <label>` when `display.relationLabels` | `<protocol> · <technology> · <payload> · transport encrypted/unencrypted`, each by its display flag |
| Sequence message | `<n>. ` when `display.stepNumbers`, then `<label>`, then ` (<protocol>)` when `display.protocols` | — |
| Sequence note | `<label>` | — |

`<n>` is the one-based index among messages in scenario order; notes are not numbered. Each line is wrapped greedily at whitespace to a framework-owned maximum width using the pinned font metrics, with grapheme-boundary fallback for overlong runs; text is never truncated.

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
├── svg-logo-profile-v1.md
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

A guideline of 7–9 main elements per level is documented in `rules/layout-rules.md`. It MUST NOT be a universal validation error. An automated warning for it is post-MVP. Diagram family, nesting and audience determine acceptable complexity.

---

## 15. Validation

Validation is part of the first vertical slice, not a late implementation phase.

### 15.1. Validation layers

The full pipeline is: safe YAML loading → config/theme/model/view structural checks → cross-document semantics → selection and capabilities → presentation/asset policy → family IR → D2 lint → `d2 validate` → rendering → decoration, legend and accessibility composition → canonicalization and mandatory SVG safety/integrity checks → manifest schema. Visual-quality review is a separate check. The technical specification section 8 decomposes these same stages; command-specific boundaries are defined in section 17 and [ADR-0017](adrs/0017-command-stage-matrix.md).

### 15.2. Required semantic checks

- unique and valid IDs,
- valid source, target, parent and scenario references,
- note participants reference elements rather than boundaries,
- no containment cycles,
- `subsystem` boundaries have the system in their ancestor chain,
- allowed element and boundary kinds,
- allowed relation-property combinations,
- allowed scenario step kinds, their required fields and uniquely identified ordered steps,
- view family/subtype compatibility,
- valid selection fields and traversal limits,
- compatibility of layout options with the selected engine,
- valid Project Configuration, theme ID, theme version and brand asset references,
- supported decoration positions, mutually exclusive slots and footer-template grammar,
- source and presentation text length and plain-text requirements (sections 8.4 and 10.6),
- no `subtitle` without `title`,
- no per-view visual overrides or raw visual styles in System Models and View Specifications,
- Theme/Brand Pack token values and semantic mappings conform to the theme schema,
- no disallowed remote assets,
- logo sanitizer-profile compliance and asset bounds below global security limits,
- no empty mandatory labels.

### 15.3. Diagnostics

Every diagnostic MUST include:

- stable diagnostic code,
- severity (`error`, `warning` or `info`),
- source file, one-based line/column and JSON Pointer when associated with an input location (optional for process, generated-output or internal failures),
- a failure category used for exit-status mapping,
- human-readable explanation,
- suggested correction when practical.

Example:

```text
FFM102 error system-model.yaml:42:9 /relations/2/target
Unknown element ID "order-db".
```

Warnings MUST NOT change generated semantics automatically.

JSON-mode diagnostics conform to the versioned `flowframe-diagnostics/v1` schema. Diagnostic file paths are relative to the resolved project root (for `validate --theme`, to the theme directory); absolute paths appear only inside optional `debug` objects. Ordering, prefixes and ambiguous category assignments are defined in [ADR-0016](adrs/0016-diagnostics-contract.md); `docs/diagnostics.md` is the code registry.

Unexpected internal failures use the `FFX` diagnostic family and exit code 6. They MUST NOT be presented as invalid user input.

---

## 16. Rendering, portability and security

### 16.1. Reproducibility

`manifest.json` MUST conform to a versioned schema. Its minimum v1 contract is:

```json
{
  "schemaVersion": "flowframe-manifest/v1",
  "flowframeVersion": "0.1.0",
  "source": {
    "config": { "path": "flowframe.yaml", "schemaVersion": "flowframe-config/v1", "sha256": "..." },
    "model": { "path": "system-model.yaml", "schemaVersion": "flowframe-model/v1", "sha256": "..." },
    "view": { "path": "infrastructure-view.yaml", "schemaVersion": "flowframe-view/v1", "sha256": "..." }
  },
  "renderer": {
    "d2Version": "...",
    "d2Sha256": "...",
    "layoutEngine": "elk",
    "layoutEngineVersion": "...",
    "flags": []
  },
  "theme": { "id": "company-light", "version": "1", "origin": "project", "sha256": "..." },
  "assetProcessing": {
    "svgSanitizer": {
      "profile": "flowframe-svg-logo/v1",
      "implementation": "...",
      "version": "..."
    },
    "assets": [
      { "kind": "logo", "id": "company-logo", "inputSha256": "...", "sanitizedSha256": "..." }
    ]
  },
  "resources": [
    { "kind": "font", "id": "flowframe-default/regular", "packageId": "flowframe", "packageVersion": "0.1.0", "license": "resource:fonts/flowframe-default/LICENSES.md", "sha256": "..." },
    { "kind": "icon", "id": "generic/database", "packageId": "flowframe", "packageVersion": "0.1.0", "license": "resource:icons/generic/LICENSES.md", "sha256": "..." }
  ],
  "outputs": {
    "d2": { "path": "diagram.d2", "sha256": "..." },
    "svg": { "path": "diagram.svg", "sha256": "..." }
  }
}
```

Source paths are relative to the resolved project root; output paths are relative to the manifest's directory. Model, View and configuration paths MUST stay within the project root; AI review sources are external read-only evidence, not compiler build inputs. Built-in resources use package resource IDs instead of host paths. Hashes MUST use SHA-256. `layoutEngineVersion` MAY be `null` only when the engine does not expose a version. `generatedAt` is optional and omitted by default. If `SOURCE_DATE_EPOCH` is supplied as a nonnegative integer Unix timestamp representable in RFC 3339, serialize that instant as UTC with `Z`; an invalid value is usage error 2. Never use the wall clock implicitly.

When the built-in configuration is used, `source.config` MUST use `{ "builtInId": "flowframe-default", "version": "1" }` instead of a path-based record. The theme hash covers raw `theme.yaml`, license inventory, every declared asset and every custom font file, with unambiguous length-prefixed paths/bytes in sorted POSIX-relative path order; built-in font references contribute their package resource IDs and raw bytes. The exact encoding is [ADR-0020](adrs/0020-theme-aggregate-hash.md). `assetProcessing` is present exactly when Theme/Brand Pack assets (logos) have been sanitized by FlowFrame during the build; each entry carries `kind: logo`. Generic icons are pre-sanitized package resources recorded only in `resources`. Font conversion belongs to the `resources` provenance records, not to the SVG-logo sanitizer record. The required `resources` array records every used font/icon: `kind`, `id`, `sha256`, `license`, and either `path` (project-root-relative) or `packageId` plus `packageVersion`. Optional `embeddedSha256` records renderer-transformed bytes; unchanged embeddings use their source hash. Renderer flags are logical options with resource IDs or project-relative paths; runtime absolute paths MUST NOT appear.

The same inputs and pinned toolchain MUST generate byte-identical `diagram.d2`. The published `diagram.svg` is canonical: normalized, verified and then hashed, so identical inputs within the toolchain equivalence class produce byte-identical SVG, and snapshot tests compare published bytes directly ([ADR-0008](adrs/0008-canonical-svg-output.md)). Every FlowFrame SVG begins with the marker comment `<!-- Generated by FlowFrame. Do not edit. -->`.

TALA builds SHOULD use a fixed seed when the installed version supports it.

### 16.2. Offline operation

The default validation and build path MUST NOT require network access. Fonts, themes, logos, brand assets and MVP icons MUST be locally available and pinned.

### 16.3. SVG portability

The implementation MUST define and test:

- whether fonts and icons are bundled,
- whether SVG text remains accessible `<text>` backed by pinned/embedded fonts or is converted to paths for stronger visual portability,
- whether the configured logo is embedded and remains visible in each supported consumer,
- title, subtitle and footer placement without clipping diagram content,
- supported documentation renderers and browsers,
- behavior when an icon cannot be embedded,
- maximum accepted input and output sizes,
- SVG sanitization requirements for publication.

### 16.4. Input and process security

- YAML loaders MUST use a safe mode and MUST NOT construct arbitrary objects. Sources are YAML 1.2 core schema; merge keys and custom tags are rejected ([ADR-0018](adrs/0018-yaml-dialect.md)).
- YAML and D2 source imports are unsupported in v0.1; declared theme assets remain restricted to the project root.
- Generated paths MUST not escape the output directory. The one exception is the private control directory `.<name>.flowframe/` next to a build output directory (section 17.2, [ADR-0010](adrs/0010-output-publication.md)).
- Implementation-wide limits are fixed per release and not user-configurable in v0.1 ([ADR-0009](adrs/0009-global-limits.md)).
- Remote render resources MUST be rejected in v0.1; optional network AI adapters are a separate boundary.
- Theme and logo SVG assets MUST be sanitized and MUST NOT contain scripts, external references or `foreignObject` content.
- D2 execution MUST have a timeout and a bounded output size.
- CI SHOULD run rendering in an isolated environment.
- Untrusted source documentation supplied to AI MUST be treated as data, not as executable instructions.

### 16.5. Logo SVG sanitizer profile

Logo sanitization MUST use the versioned `flowframe-svg-logo/v1` profile. The implementation MUST maintain an element-and-attribute allowlist and reject unsupported rendering or active content. It MAY remove only the explicitly allowlisted inert editor metadata described below.

The exact per-element attribute and CSS-property matrix is normative and MUST be published in `rules/svg-logo-profile-v1.md`. At minimum it covers:

- geometry and coordinate attributes required by the allowed vector elements,
- safe paint attributes including fill, stroke, opacity, line and fill rules,
- transform, gradient, clipping, masking and bounded-filter attributes,
- fragment-only `href`, `url(#id)`, `id` and `class` references,
- accessibility metadata and XML namespace attributes,
- a CSS property allowlist equivalent to the permitted presentation attributes.

The v1 profile MUST support:

- vector structure and geometry: `svg`, `g`, `defs`, `title`, `desc`, `path`, `rect`, `circle`, `ellipse`, `line`, `polyline` and `polygon`,
- paint servers: `linearGradient`, `radialGradient` and `stop`,
- clipping and masking: `clipPath` and `mask`,
- internal reuse through `use` and fragment-only `href="#id"` or `xlink:href="#id"` (conflicting attributes are errors),
- internal `url(#id)` references for gradients, clipping, masks and filters,
- a bounded filter subset: `filter`, `feBlend`, `feColorMatrix`, `feComposite`, `feDropShadow`, `feFlood`, `feGaussianBlur`, `feMerge`, `feMergeNode` and `feOffset`,
- safe presentation attributes and inline style properties required by those features.

The v1 profile MUST reject:

- `script`, `foreignObject`, embedded HTML, frames, audio, video and animation elements,
- event-handler attributes such as `onload` or `onclick`,
- external URLs, network references, non-fragment `href`, CSS `@import` and external `url(...)`,
- embedded raster images and executable or active content,
- unknown elements, attributes, CSS rules or filter primitives,
- assets exceeding global byte-size, dimension, element-count, nesting or filter-region limits.

Canonicalization removes XML comments, `<metadata>` subtrees and attributes in explicitly enumerated Inkscape/Sodipodi namespace URIs after bounded parsing and whole-document active-content/URL checks. Unknown namespaces and rendering elements remain errors. This narrow inert-metadata exception is not permission to strip arbitrary unsupported content. Number serialization MUST preserve numeric value without rounding; namespace and ID rewrites preserve all valid references.

A `<style>` element MAY be accepted only when parsed by a real CSS parser against a property allowlist; regular-expression-only CSS sanitization is forbidden. Otherwise authors must convert styles to permitted presentation attributes. Rejected assets produce an actionable diagnostic and are never rendered in partially sanitized form.

For identical input bytes, sanitizer profile, implementation and version, sanitized output MUST be byte-identical after canonical namespace and attribute ordering plus deterministic ID rewriting and reference normalization. The manifest records the profile, implementation version and input/output SHA-256 for every processed brand asset.

---

## 17. CLI contract

The core v0.1 commands use named options consistently:

```text
flowframe validate --model MODEL [--view VIEW] [--config CONFIG]
flowframe validate --theme THEME_FILE
flowframe compile --model MODEL --view VIEW [--config CONFIG] --output D2
flowframe render --input D2 [--layout elk] [--config CONFIG] --output SVG
flowframe build --model MODEL (--view VIEW | --all) [--config CONFIG]
  [--layout elk] --output-dir DIR
flowframe review --model MODEL --view VIEW [--config CONFIG]
  [--source SOURCE] [--mode syntax|semantic|policy|source-conformance]
flowframe version
```

All commands except `version` accept `--format text|json` (default text) and `--debug`. Diagnostics go to stderr; JSON mode emits one complete JSON diagnostic array conforming to `flowframe-diagnostics/v1`, including an empty array on success. In text mode artifact paths may be printed to stdout; in JSON mode stdout is empty. In JSON mode debug details use an optional `debug` object inside a diagnostic; no extra log text may corrupt the JSON stream. Debug output MUST NOT expose credentials. A `--strict` warning-promotion option is deferred.

Every command uses one validation service. Which stages each command runs is defined by the command stage matrix in [ADR-0017](adrs/0017-command-stage-matrix.md); for the same inputs, `validate --model M --view V`, default `review` and `build` report identical diagnostics for every stage up to and including the family IR. In particular, `validate --view` runs selection, so empty selections, flow undirected-dependency errors and selection warnings are reported without rendering.

`validate` without `--view` checks the model, resolved configuration and theme, but skips view-specific checks. Whole-project `validate --all` is post-MVP; `build --all` is the v0.1 whole-project command. `--theme` is an exclusive form that validates the theme and declared assets without a model/config; its theme directory is the asset root. Neither form invokes D2. Full asset validation is available from P4; pre-release limitations are explicit in the plan.

`compile` runs source/asset/IR validation and policy lint, then emits one self-contained generated D2 file. It does not invoke D2 or emit a manifest.

`compile --output` and `render --output` MUST NOT resolve to any input file of the invocation. An existing output file is replaced only if it is a FlowFrame artifact of the same type (a D2 file with a valid FlowFrame header, or an SVG starting with the FlowFrame marker comment); otherwise the command fails with a policy error (5) and leaves the file unchanged ([ADR-0010](adrs/0010-output-publication.md)). Classes are generated from the resolved theme; packaged `resources/d2` files are internal templates, not output imports.

`render` is a low-level body-only operation: it requires the FlowFrame generated-file header and passes the complete D2 policy lint before `d2 validate` and rendering. A header is not proof of safety. Hand-authored D2 is unsupported. It emits one verified body SVG, with no project decorations or manifest. Optional explicit `--config` supplies the same theme/font environment used at compile time; without it only the built-in configuration is used and no upward discovery occurs. The generated header includes a deterministic theme fingerprint, exactly the aggregate SHA-256 value recorded as `theme.sha256` in a build manifest (the same raw-byte hashing algorithm, even when compile emits no manifest). It is not a hash of D2 contents or an authenticity proof. A missing or malformed header and a mismatch with the resolved render theme are validation errors (1). `render --layout` overrides `render.layoutEngine` from an explicit `--config` with the same precedence as `build`. Therefore `compile + render` is not equivalent to `build`. This restriction avoids a hidden dependency on compiler working directories.

`build` performs the complete pipeline, including `d2 validate`, decorations, accessibility summary, output verification and manifest publication. For a single `--view`, outputs are directly in DIR. `--all` requires a nonempty explicit config `views` list. Before rendering the first view it runs every validation stage up to the family IR for all entries; any error there stops the command with nothing published. It then renders and writes each view to `DIR/<view-id>/` in sorted View-ID order. Both view options together, neither option, an empty registry or duplicate output IDs are errors. All entries share the supplied model, configuration and theme. Each view is prepared completely before publication to its ordinary output directory, using the recoverable replacement contract in section 17.2. Directory replacement is not atomic for concurrent readers, and the project is not one transaction; a render or publication failure stops processing and reports which views were published.

D2 executable resolution uses `FLOWFRAME_D2` if set (an absolute executable path; a relative path is usage error 2), otherwise the first `d2` on PATH. The pin is the package resource `d2-pin.json`; the installer helper is `python -m flowframe.install_d2` ([ADR-0006](adrs/0006-pinned-d2-distribution.md)). Every render invocation MUST verify version and SHA-256 against the packaged platform-specific pin; a mismatch or absence is dependency error 3. There is no separate reproducible mode: the same rule applies locally and in CI. Supported executables are the official release binaries accepted by the packaged per-platform pin. A distribution rebuild is unsupported unless its bytes match that pin; matching the version string alone is insufficient. FlowFrame MUST provide an explicit installer helper and offline installation instructions for the approved artifact, without downloading it implicitly during a build. Diagnostics distinguish executable missing, version mismatch and checksum mismatch and point to those instructions. Updating the approved D2 pin requires a FlowFrame release.

`review` is read-only. MVP modes `syntax`, `semantic` and `policy` reuse the same validation pipeline, diagnostic codes and source-location handling as `validate`, without D2 or AI; they MUST NOT maintain a second implementation of those rules. The default review modes produce the same findings/status as `validate --model ... --view ...` for the same inputs. Explicit modes select validation stages while retaining prerequisite checks; syntax here means YAML/schema correctness, not rendering. `--mode` and `--source` are repeatable. Default modes are syntax, semantic and policy. Optional `source-conformance` requires at least one source and the P7 AI adapter. Missing requested adapter returns 3. Invalid combinations return 2: `--source` without `source-conformance`, and `source-conformance` without `--source`. Source-conformance findings use the `FFA` prefix with severity `warning` and do not change the exit status. Visual review and its future `--svg` option are post-MVP.

The post-MVP layout comparison command has this provisional contract:

```text
flowframe compare-layouts --model MODEL --view VIEW [--config CONFIG]
  --layout ENGINE --layout ENGINE [--output-dir build/layouts/]
```

It validates and compiles the model/view once, then renders the same generated D2 with each explicitly named installed engine. It MUST NOT silently substitute an engine. Results are written below `<output-dir>/<engine>/`. The command requires at least two distinct engines and remains post-MVP while ELK is the only required engine. Fewer than two distinct engines is usage error 2; an unavailable engine returns 3; any engine render failure returns 4 and prevents a successful comparison result.

### 17.1. Exit codes

```text
0  success
1  validation or compilation error
2  invalid command usage
3  missing external dependency
4  execution failure: renderer, timeout, output I/O or publication conflict
5  policy or security violation
6  unexpected internal failure
130 interrupted by SIGINT
```

Each error diagnostic carries one category; prefixes identify areas, not exit codes:

| Category | Typical prefixes/cases | Exit |
|---|---|---:|
| validation | FFC/FFS/FFM/FFV/FFT/FFI/FFD: malformed source, bad references, invalid combination, unsupported engine in configuration, source file outside the project root, missing/malformed D2 header or theme fingerprint mismatch, supplied D2 artifact rejected by standalone render | 1 |
| usage | invalid CLI combinations, unsupported `--layout` value, invalid `SOURCE_DATE_EPOCH` or relative `FLOWFRAME_D2` | 2 |
| dependency | FFR: missing or wrong pinned renderer; FFA: missing requested adapter | 3 |
| execution | FFR: render process failure, timeout, invalid output geometry, output I/O, occupied writer lock or failed publication/recovery | 4 |
| policy | FFC/FFS/FFT/FFD/FFR: forbidden styling fields, remote resources, symlink or asset path escape, forbidden output directory, refused overwrite of a non-artifact file, input or SVG exceeding fixed limits, unsafe SVG | 5 |
| internal | FFX: unexpected implementation failure, including syntax rejection of D2 freshly generated within build | 6 |

A syntax rejection from `d2 validate` inside build is FFX/internal (6), because FlowFrame just generated the input under its pinned toolchain. In standalone render, a supplied artifact can be corrupted or edited despite its header, so a syntax rejection is FFD/validation (1). Failure to execute D2 uses dependency (3), while a process crash or timeout uses execution (4). An occupied writer lock has its own stable FFR diagnostic and returns 4 immediately; it is not invalid model input.

Known forbidden styling fields produce policy errors even if rejected by a closed schema; other unknown fields are ordinary validation errors. Structural schema checks may stop before dependent policy checks. Among errors actually observed, precedence is 6 > 5 > 4 > 3 > 2 > 1; usage errors normally terminate before processing. SIGINT terminates with 130 after cleanup. Warnings/info alone return 0. Never continue unsafe processing merely to discover a higher-priority error.

### 17.2. Build output

```text
build/
├── diagram.d2
├── diagram.svg
└── manifest.json
```

The CLI MUST write diagnostics to stderr and MUST NOT infer success from the presence of an output file. It MUST check the D2 process exit status because D2 may leave a partial output after a rendering error. The logical per-view output is an ordinary dedicated directory containing the three artifacts. Builds prepare and verify a complete staging directory before replacing the target under a writer lock (technical-spec §11.1). An existing empty directory is accepted; a previous FlowFrame output is replaceable only when its manifest (in any manifest version the release can read) and file inventory identify the expected intact artifact set. Unrelated files, modified artifacts or output symlinks are rejected with an actionable diagnostic and are not overwritten.

Besides the target, a build writes only the private control directory `<parent>/.<name>.flowframe/` (lock, staging, one backup, recovery journal). It is untracked build state. `--output-dir` is rejected (policy, 5) when it is a filesystem root, the process working directory, the resolved project root, an ancestor of any resolved source, configuration, theme or asset file, inside a theme directory, or a symlink. Only local POSIX filesystems are supported ([ADR-0010](adrs/0010-output-publication.md)).

Publication renames the previous output to a bounded backup, then staging to the target. This is recoverable replacement, not an atomic directory swap: readers can briefly observe a missing directory and concurrent multi-file reads have no snapshot guarantee. Documentation builds and copies for publication SHOULD consume outputs after a successful build finishes. A render/validation failure before publication leaves the old output unchanged. A publication failure attempts rollback; if rollback itself fails, the command reports the backup/recovery location and never claims the old output has been restored.

Retain at most one previous output and one in-progress staging set per target; automatic recovery/cleanup on the next invocation, under the lock, removes only verified FlowFrame-owned state. There is no history of generations, content-addressed output naming, separate export command or symlink publication mode in v0.1. Final SVG/D2/manifest files can be linked, copied or committed directly; private staging/backup state is not part of the published artifacts.

---

## 18. AI integration

AI is an optional adapter around the deterministic core. Network-backed adapters are explicitly outside the offline-core guarantee and cannot run implicitly during validate, compile or build. The adapter is not part of the core CLI contract: it ships as an optional extra or agent skill that consumes the public contracts and writes only model/View files plus an `assumptions.md` report. Select the first adapter platform at P7.1 and freeze the evaluation corpus and gating thresholds at P7.4 before the release evaluation run ([ADR-0025](adrs/0025-ai-adapter-boundary.md)).

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
- claim that a model matches a real system without a source-conformance check,
- follow instructions embedded in supplied source documents.

### 18.2. Review modes

Review distinguishes the following scopes; the deterministic syntax/semantic/policy modes and optional source-conformance are in v0.1:

- **syntax review** — YAML and schema correctness (D2 validation belongs to render/build),
- **semantic review** — internal consistency of the model,
- **source-conformance review** — comparison with supplied source material,
- **visual review (post-MVP)** — readability of a rendered SVG/PNG,
- **policy review** — compliance with FlowFrame rules.

Without source material, the reviewer cannot determine whether the documented architecture is factually correct.

### 18.3. Improvement loop

This loop and the `simplify-view`/diagram-refactor adapter are post-MVP. Before implementing it, a separate ADR MUST define the quality metric and stopping threshold. An automated improvement loop MAY adjust only View Specification and layout options. It MUST:

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
External client → Internet → Application Gateway → Kubernetes API → PostgreSQL.
The Internet labels the client's route to the gateway, not a traffic endpoint;
the API-to-PostgreSQL connection is internal.
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

The canonical detailed package layout is in [technical-spec.md §5](technical-spec.md#5-proposed-package-layout). Tests use `tests/unit`, `contract`, `integration`, `golden`, `snapshots` and `security`.

The shared example project lives in `examples/payments/`, with its own `flowframe.yaml`, `system-model.yaml` and three View files. The implementation repository MUST NOT have an example `flowframe.yaml` in its root. Consumer themes live in `<project-root>/themes/<theme-id>/`; built-in schemas, themes, default mappings, icons and D2 internal templates are installed package resources only.

Architecture decisions are recorded in `docs/adrs/` ([register](adrs/README.md)); the diagnostic code registry is `docs/diagnostics.md`. `rules/` contains the normative companions listed in section 14; `prompts/` and `skills/` are optional AI adapters, with only the P7 generation/review workflow in MVP. Single-view output uses the caller's output directory; multi-view output uses one subdirectory per View ID.

---

## 21. Delivery stages

Detailed tasks and gates are maintained only in [implementation-plan.md](implementation-plan.md).

| Product stage | Plan phases | Release result |
|---|---|---|
| Stage 0 — Decisions/spikes | P0 | Accepted ADRs and measured external behavior |
| Stage 1 — Contract-first slice | P1–P3 | Infrastructure alpha.1 with built-in presentation only |
| Stage 2 — Design system | P4, hardened in P6 | Full global branding, icons and accessibility |
| Stage 3 — Remaining families | P5, hardened in P6 | Flow and sequence from the same model |
| Stage 4 — AI/review | P7 | Optional adapter and read-only review |
| Stage 5 — Distribution | P6, P8 | Verified v0.1 release |
| Stage 6 — Extensions | P9 | Post-MVP capabilities |

---

## 22. MVP acceptance criteria

FlowFrame v0.1 is accepted when all of the following are true:

1. All example configurations, models, views and themes pass schema and semantic validation.
2. Invalid references, containment cycles, brand assets and unsupported values produce stable, actionable diagnostics.
3. The same configuration, model, view and pinned toolchain generate byte-identical D2, and byte-identical canonical `diagram.svg` within the toolchain equivalence class, when built twice in different absolute paths.
4. All MVP examples render offline with ELK in local and CI environments.
5. Every diagram in one project uses the same resolved Theme/Brand Pack, and per-view visual overrides are rejected.
6. A custom brand fixture consistently renders its global color mappings, embedded logo, title and source footer across all MVP families.
7. Generated D2 contains no raw styles originating from System Model or View Specification files.
8. Infrastructure, integration-flow and sequence examples are produced from the same reusable System Model where applicable.
9. In the golden corpus, automated bounds checks confirm that decoration and legend bounds do not intersect the body bounds, every label's text bounds lie inside its shape, and sibling node bounds do not intersect; a recorded human review finds no clipped or overlapping content.
10. Grayscale readability: every pair of element kinds differs by shape or border style, every pair of relation interactions differs by stroke pattern, and every relation shows its semantic name as text (sections 13.5, 13.9, 13.10); a recorded human review of grayscale snapshots finds no meaning conveyed by color alone.
11. The built-in and custom themes pass every contrast pair of section 13.9 (4.5:1 text, 3:1 essential non-text).
12. The manifest records configuration, theme and input hashes together with renderer and sanitizer profiles, versions and options needed to reproduce the build.
13. Renderer failures and timeouts return documented non-zero exit codes and do not report success based on partial output files.
14. The AI adapter meets every gating metric of [ADR-0025](adrs/0025-ai-adapter-boundary.md) on the frozen corpus, including zero styling fields and a fully passing prompt-injection corpus.
15. One documented `build --all` command builds the registered example views from source.
16. The security corpus fails closed, including YAML limits, path escape, unsafe SVG/CSS and renderer output.
17. Every `build` SVG includes accessible title/description/object metadata and the deterministic textual summary of section 13.7; visible legends explain non-color relation styles.
18. Read-only syntax/semantic/policy review works without AI; source-conformance works with the optional adapter. Diagnostics, categories and all exit codes match section 17, and JSON output validates against `flowframe-diagnostics/v1`.
19. A render or validation failure never modifies the previous output; a failed publication either restores the previous output or reports the backup location; forbidden output directories and non-artifact overwrites are refused.
20. The D2 installer helper installs the pinned executable from a local archive without network access; missing executable, wrong version and wrong checksum produce distinct exit-3 diagnostics.
21. Small and medium corpus builds stay within the performance budgets of [ADR-0023](adrs/0023-performance-budgets.md).

P0 sets render-performance budgets from measured baselines. P7.4 freezes the AI corpus and gating thresholds before the release evaluation run; neither may be relaxed after seeing the release results.

---

## 23. Quality and test strategy

### 23.1. Test categories

- JSON Schema positive and negative fixtures for Project Configuration, Theme/Brand Pack, System Model, View Specification and manifest,
- semantic-validator unit tests,
- projection tests for every family,
- deterministic D2 golden tests,
- renderer integration tests,
- normalized SVG snapshots,
- offline-build tests,
- accessibility checks,
- global-theme consistency tests across multiple views,
- title, subtitle, footer and logo snapshot tests,
- footer-template tests covering static text, `{source}`, escaped braces and invalid placeholders,
- sanitizer allowlist, rejection, complexity-limit and deterministic-output tests,
- CLI exit-code tests,
- security tests for unsafe YAML, path traversal, unsafe SVG brand assets and remote assets,
- AI evaluations separated from deterministic core tests.

### 23.2. Golden corpus

The MVP corpus SHOULD contain at least:

- five infrastructure views,
- five integration-flow views,
- five sequence views,
- small, medium and boundary-case diagrams,
- nested boundaries,
- intentionally absent icons (`null`) and invalid packs with missing declared assets,
- long labels and Unicode labels,
- a custom global theme with title, source footer and embedded logo,
- views with omitted optional presentation fields,
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
| Unsafe or unlicensed brand assets | security or legal exposure | validated local assets, SVG sanitization, mandatory license metadata |
| Sanitizer changes alter embedded logos | non-reproducible output | versioned profile, pinned implementation, canonical output and manifest hashes |
| Decorations overlap diagram content | unreadable diagrams | measured canvas bands, slot-conflict validation and visual snapshots |
| Per-view branding drifts from project identity | inconsistent documentation | one project configuration; reject per-view visual overrides |
| D2 classes are overridden | inconsistent styling | generated D2 only, linting, no style fields in source schemas |
| Automatic review changes semantics | architecture corruption | immutable System Model during layout improvement, visible diff |
| Sequence requirements outgrow static relations | incompatible model | explicit ordered scenarios and Sequence IR |
| Large diagrams remain unreadable | poor usability | view filtering, warnings, future aggregation |

---

## 25. Open decisions

Open decisions and open parameters are tracked only in the ADR register [`docs/adrs/README.md`](adrs/README.md). Every ADR assigned to a P0 phase MUST be Accepted before the P0 exit gate; open parameters of Accepted ADRs are filled by the phase named in the ADR. Changing a decision that affects this PRD requires a superseding ADR and a PRD update in the same change.

Python packaging (`uv` with Hatchling) is decided in [ADR-0002](adrs/0002-python-cli-and-d2-process-boundary.md). AI platform selection belongs to P7.1 and evaluation thresholds to P7.4 ([ADR-0025](adrs/0025-ai-adapter-boundary.md)).

Dark theme support is not an open v0.1 decision: it remains post-MVP as defined in section 3.2.

---

## 26. First milestone

The first practical milestone is a contract-first vertical slice:

```text
FlowFrame v0.1-alpha.1
```

It MUST accept:

```text
examples/payments/flowframe.yaml
examples/payments/system-model.yaml
examples/payments/infrastructure-view.yaml
```

and produce with one command:

```text
build/payments/infrastructure-overview/
├── diagram.d2
├── diagram.svg
└── manifest.json
```

Alpha.1 uses an explicit minimal configuration with `theme: flowframe-light` and `branding.footer.enabled: false`, a View with `display.legend: false`, and no logo/custom theme or view presentation metadata. It rejects not-yet-supported presentation features with an actionable validation diagnostic marked `prerelease-only` in the diagnostic registry; it MUST NOT silently ignore them. The output directory name equals the View ID, matching the `build --all` layout. P3 implements the built-in theme hash and omits `assetProcessing` when no vector asset is sanitized. Full branding, accessibility composition and sanitizer checks arrive in P4. Reuse of the same model for flow and sequence is the P5 gate, not an alpha.1 acceptance condition.
