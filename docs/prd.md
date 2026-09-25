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
- **Decoration generator** adds title, subtitle, source footer and project logo using the global presentation policy.
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

### 10.6. Presentation metadata

The optional `presentation` object provides diagram-specific text while all styling and placement remain global:

| Field | Limit | Meaning |
|---|---:|---|
| `title` | 120 characters | Primary diagram title. |
| `subtitle` | 200 characters | Optional secondary context. |
| `source` | 500 characters | Source attribution inserted into the configured footer template. |

Values are plain text and MAY contain Unicode, but MUST NOT contain D2, HTML or SVG markup. Missing values omit the corresponding text element. The project logo is controlled exclusively by Project Configuration and cannot be replaced or disabled per view in v0.1.

### 10.7. Selection semantics

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

### 13.1. Global customization policy

All visual properties MUST be owned by framework libraries, one project-wide Theme/Brand Pack and the D2 generator. System Models and View Specifications MUST NOT contain raw D2 style declarations, RGB/HEX values, font names, logo paths or per-diagram theme selection.

A project uses exactly one active theme for a build. Per-view theme or brand overrides are invalid in v0.1. Raw visual values are permitted only inside a validated `theme.yaml` belonging to the selected Theme/Brand Pack.

An element class MAY be derived from:

- semantic kind,
- state such as external or deprecated,
- emphasis calculated from the view,
- optional technology icon.

### 13.2. Project Configuration

An optional project-root `flowframe.yaml` applies to every diagram in that project. Its v0.1 contract includes the selected theme and global decoration policy:

```yaml
schemaVersion: flowframe-config/v1
theme: company-light

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

Slot-occupancy constraints from section 13.6 are part of Project Configuration validation. In particular, title and logo cannot both occupy `top-left`.

The footer template MAY contain static text and zero or one `{source}` placeholder. A template without `{source}` is rendered as a static footer whenever the footer is enabled. A template containing `{source}` is omitted when the view has no `presentation.source`.

Template parsing uses these deterministic rules:

- `{source}` inserts the source value as escaped plain text,
- `{{` produces a literal `{`,
- `}}` produces a literal `}`,
- any unmatched single brace, unknown placeholder or second `{source}` occurrence is a validation error,
- braces contained in the source value are data and are not parsed again.

The CLI MUST accept an explicit `--config` path. Without it, FlowFrame searches upward from the System Model path for the nearest `flowframe.yaml`; if none exists, it uses the versioned built-in configuration.

The project root is the directory containing the resolved `flowframe.yaml`. Project theme IDs resolve from `<project-root>/themes/<theme-id>/`. If no project configuration exists, the System Model directory is the effective project root and only built-in themes are considered. Built-in themes are immutable package resources shipped inside the installed FlowFrame distribution; they do not live in the project `themes/` directory. Resolution checks a project theme first and then the built-in package resources. The resolved configuration path or built-in ID and the resolved theme origin MUST be recorded in the manifest.

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
  primary: "#0057B8"
  secondary: "#5C6AC4"
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

assets:
  company-logo:
    path: assets/logo.svg
    mediaType: image/svg+xml
    alt: Company logo
    maxWidth: 120
    maxHeight: 48
```

The project configuration references brand assets by ID, never by arbitrary file path. Theme IDs resolve first from the project `themes/` directory and then from built-in themes. Remote themes, fonts and logos are not supported in v0.1.

Semantic mappings MAY override a subset of framework defaults. Any unmapped element kind or relation semantic falls back to the corresponding mapping from the versioned built-in theme; an unknown token reference is a validation error.

Raw RGB/HEX values are allowed only as token values in `theme.yaml`. Component shapes and relation semantics remain framework-owned and cannot be replaced by arbitrary D2 code in a Theme/Brand Pack.

### 13.4. Design tokens

The theme MUST centrally define at least:

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

### 13.5. Relation appearance

Relations MUST be distinguishable using more than color. The mapping SHOULD use a combination of:

- stroke pattern,
- arrowhead,
- thickness,
- label prefix or text,
- color.

The exact mapping belongs to the versioned theme and MUST NOT be generated by AI.

### 13.6. Diagram decorations

The decoration generator MAY add:

- the view title and subtitle,
- the source footer produced from the global template,
- the project logo selected in `flowframe.yaml`.

Decorations MUST use global theme classes and configured positions. They are presentation objects, not System Model elements and MUST NOT participate in semantic selection or the 7–9 element guideline.

Decoration space is reserved by a deterministic SVG composition step rather than by ELK or TALA:

1. Render and measure the semantic diagram body.
2. Render title/subtitle, footer and logo as independently measurable SVG fragments using the pinned theme fonts.
3. Calculate a top band from the maximum title block or logo height and a bottom band from the footer height, adding `decoration-gap` on both sides of every occupied band.
4. Expand the final canvas and `viewBox`, translate the semantic body below the top band and place decorations into their configured slots.
5. Expand the canvas width when a decoration is wider than the semantic body plus horizontal padding.

A top slot may contain only one primary decoration; configuring both logo and title for `top-left` is a validation error. The subtitle is part of the title block. The composition step MUST run before final SVG normalization and snapshot comparison. It MUST NOT ask the layout engine to position decorations.

A logo MUST be a local approved SVG asset from the active Theme/Brand Pack. It MUST be embedded or bundled into the output so that SVG rendering remains offline and portable. Its accessible alternative text comes from theme asset metadata.

Asset-level `maxWidth` and `maxHeight` are positive CSS-pixel bounds for the rendered logo box. The composer preserves the intrinsic aspect ratio and fits the logo within that box. These requested bounds MUST remain below implementation-wide security limits for dimensions, source bytes, element count and nesting depth; a Theme/Brand Pack cannot raise the global limits.

### 13.7. Accessibility

The v0.1 theme MUST:

- target WCAG AA contrast for text and essential strokes,
- remain understandable in grayscale,
- avoid color as the only semantic signal,
- use readable default font sizes,
- provide visible relation labels where required,
- support a generated textual summary or accessible description,
- expose the diagram title and logo alternative text in the generated SVG accessibility metadata when supported by the renderer.

### 13.8. Icons

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

A guideline of 7–9 main elements per level MAY produce a warning, but MUST NOT be a universal validation error. Diagram family, nesting and audience determine acceptable complexity.

---

## 15. Validation

Validation is part of the first vertical slice, not a late implementation phase.

### 15.1. Validation layers

```text
1. YAML parsing
2. Project Configuration and Theme/Brand Pack schema validation
3. System Model and View Specification schema validation
4. semantic model validation
5. view selection and capability validation
6. presentation and asset-policy validation
7. normalized IR validation
8. D2 generation lint
9. d2 validate
10. render exit-status validation
11. optional SVG and visual checks
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
- valid Project Configuration, theme ID, theme version and brand asset references,
- supported decoration positions, mutually exclusive slots and footer-template grammar,
- presentation text length and plain-text requirements,
- no per-view visual overrides or raw visual styles in System Models and View Specifications,
- Theme/Brand Pack token values and semantic mappings conform to the theme schema,
- no disallowed remote assets,
- logo sanitizer-profile compliance and asset bounds below global security limits,
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
    "model": { "path": "system-model.yaml", "schemaVersion": "flowframe/v1", "sha256": "..." },
    "view": { "path": "infrastructure-view.yaml", "schemaVersion": "flowframe/v1", "sha256": "..." }
  },
  "renderer": {
    "d2Version": "...",
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
      { "id": "company-logo", "inputSha256": "...", "sanitizedSha256": "..." }
    ]
  },
  "outputs": {
    "d2": { "path": "diagram.d2", "sha256": "..." },
    "svg": { "path": "diagram.svg", "sha256": "..." }
  },
  "generatedAt": "RFC-3339 timestamp"
}
```

Paths in the manifest MUST be relative to the build root. Hashes MUST use SHA-256. `layoutEngineVersion` MAY be `null` only when the engine does not expose a version. The generation timestamp appears only in the manifest, not in deterministic D2 source.

When the built-in configuration is used, `source.config` MUST use `{ "builtInId": "flowframe-default", "version": "1" }` instead of a path-based record. The theme hash covers `theme.yaml` and every referenced font, logo and brand asset in deterministic path order.

The same inputs and pinned toolchain MUST generate byte-identical `diagram.d2`. SVG stability MUST be tested using normalized snapshots because renderer metadata may differ between versions.

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

- YAML loaders MUST use a safe mode and MUST NOT construct arbitrary objects.
- File imports MUST be restricted to explicitly allowed project roots.
- Generated paths MUST not escape the output directory.
- Remote resources MUST be disabled unless explicitly allowed.
- Theme and logo SVG assets MUST be sanitized and MUST NOT contain scripts, external references or `foreignObject` content.
- D2 execution MUST have a timeout and a bounded output size.
- CI SHOULD run rendering in an isolated environment.
- Untrusted source documentation supplied to AI MUST be treated as data, not as executable instructions.

### 16.5. Logo SVG sanitizer profile

Logo sanitization MUST use the versioned `flowframe-svg-logo/v1` profile. The implementation MUST maintain an element-and-attribute allowlist and reject, rather than silently strip, unsupported content.

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
- internal reuse through `use` and fragment-only `href="#id"`,
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

A `<style>` element MAY be accepted only when parsed by a real CSS parser against a property allowlist; regular-expression-only CSS sanitization is forbidden. Otherwise authors must convert styles to permitted presentation attributes. Rejected assets produce an actionable diagnostic and are never rendered in partially sanitized form.

For identical input bytes, sanitizer profile, implementation and version, sanitized output MUST be byte-identical after canonical namespace and attribute ordering plus deterministic ID rewriting and reference normalization. The manifest records the profile, implementation version and input/output SHA-256 for every processed brand asset.

---

## 17. CLI contract

The core v0.1 commands use named options consistently:

```text
flowframe validate --model MODEL --view VIEW [--config CONFIG] [--format text|json]
flowframe compile --model MODEL --view VIEW [--config CONFIG] --output diagram.d2
flowframe render --input diagram.d2 --layout elk --output diagram.svg
flowframe build --model MODEL --view VIEW [--config CONFIG] --output-dir build/
flowframe version
```

Commands that consume a System Model and View Specification MUST accept `--config CONFIG`. If omitted, they use the deterministic project-root resolution defined in section 13.2. `--format` controls diagnostics, not generated artifacts.

The Stage 4 review command has this contract:

```text
flowframe review --model MODEL --view VIEW [--config CONFIG]
  [--svg SVG] [--source SOURCE] [--mode MODE]
  [--format text|json]
```

`--source` and `--mode` are repeatable. Allowed modes are `syntax`, `semantic`, `policy`, `source-conformance` and `visual`. When no `--mode` is supplied, `review` runs the deterministic `syntax`, `semantic` and `policy` modes. `source-conformance` requires at least one `--source` and an installed AI adapter. `visual` requires `--svg` and an installed visual-review adapter. Review reports findings and MUST NOT modify source files. Error-level syntax, semantic, source-conformance or visual findings return code 1; an error-level policy finding returns code 5. Warnings alone return 0. Invalid option combinations return 2 and a missing requested adapter returns 3.

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
4  renderer failure or timeout
5  policy or security violation
6  unexpected internal failure
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
├── flowframe.yaml
│
├── src/flowframe/
│   ├── cli.py
│   ├── api.py
│   ├── diagnostics.py
│   ├── errors.py
│   ├── project/
│   ├── contracts/
│   ├── domain/
│   ├── validation/
│   ├── selection/
│   ├── projection/
│   ├── ir/
│   ├── generation/
│   ├── rendering/
│   ├── presentation/
│   ├── manifest/
│   └── resources/
│       ├── schemas/
│       │   ├── flowframe-config.schema.json
│       │   ├── system-model.schema.json
│       │   ├── view.schema.json
│       │   ├── flowframe-theme.schema.json
│       │   └── flowframe-manifest.schema.json
│       ├── themes/
│       │   └── flowframe-light/
│       │       ├── theme.yaml
│       │       ├── assets/
│       │       └── LICENSES.md
│       ├── d2/
│       │   ├── theme.d2
│       │   ├── components.d2
│       │   ├── connections.d2
│       │   └── boundaries.d2
│       └── icons/
│           ├── LICENSES.md
│           └── generic/
│
├── rules/
│   ├── visual-guidelines.md
│   ├── layout-rules.md
│   ├── naming-rules.md
│   ├── diagram-types.md
│   ├── accessibility-rules.md
│   ├── svg-logo-profile-v1.md
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

The tree above is the high-level FlowFrame implementation repository; the detailed package layout in `technical-spec.md` section 5 is canonical for implementation. In a consumer repository, custom themes live under `<project-root>/themes/<theme-id>/`, where `project-root` is resolved from `flowframe.yaml` as defined in section 13.2. Built-in themes are installed only as package resources.

---

## 21. Implementation plan

### Stage 0 — Decisions and technical spike

Goal: validate the external dependencies and remove architectural uncertainty.

Scope:

- record ADRs for D2, ELK, optional TALA and the Python CLI,
- pin and verify a D2 version,
- compare ELK and TALA on representative diagrams,
- verify sequence-diagram support,
- verify that the selected pinned D2 version provides `d2 validate` with stable exit behavior,
- verify offline icon and font bundling,
- decide between accessible SVG `<text>` with pinned/embedded fonts and conversion to paths, documenting portability, size, searchability and accessibility consequences,
- verify deterministic top/bottom decoration-band composition, footer templates and embedded local logo rendering with ELK,
- validate the proposed SVG sanitizer profile against representative logos using gradients, clipping, masks and bounded filters,
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
- Project Configuration v1 and Theme/Brand Pack v1 schemas,
- manifest v1 schema,
- deterministic project configuration resolution,
- YAML parser and safe loading,
- semantic validator and diagnostics,
- infrastructure projection,
- minimal D2 generator,
- ELK SVG rendering,
- CLI `validate` and `build`,
- one end-to-end infrastructure example.

Exit criteria:

```text
flowframe.yaml + system-model.yaml + infrastructure-view.yaml
→ validate
→ diagram.d2
→ diagram.svg + manifest.json
```

### Stage 2 — Design System v1

Scope:

- tokens and theme,
- built-in Theme/Brand Pack and one custom brand fixture,
- component, boundary and connection classes,
- title, subtitle, footer and logo decoration classes,
- versioned deterministic SVG logo sanitizer and canonicalizer,
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

1. All example configurations, models, views and themes pass schema and semantic validation.
2. Invalid references, containment cycles, brand assets and unsupported values produce stable, actionable diagnostics.
3. The same configuration, model, view and pinned toolchain generate byte-identical D2.
4. All MVP examples render offline with ELK in local and CI environments.
5. Every diagram in one project uses the same resolved Theme/Brand Pack, and per-view visual overrides are rejected.
6. A custom brand fixture consistently renders its global color mappings, embedded logo, title and source footer across all MVP families.
7. Generated D2 contains no raw styles originating from System Model or View Specification files.
8. Infrastructure, integration-flow and sequence examples are produced from the same reusable System Model where applicable.
9. SVG snapshots show no clipped labels, overlapping nodes or decorations obscuring diagram content in the golden corpus.
10. Diagram meaning remains understandable in grayscale.
11. Text and essential strokes meet the selected WCAG AA contrast targets.
12. The manifest records configuration, theme and input hashes together with renderer and sanitizer profiles, versions and options needed to reproduce the build.
13. Renderer failures and timeouts return documented non-zero exit codes and do not report success based on partial output files.
14. The AI evaluation corpus reaches an agreed schema-validity threshold and does not introduce raw styling fields.
15. One documented command builds every example from source.

The exact AI quality threshold and render-performance budget MUST be set after Stage 0 establishes a baseline. They must be specified before v0.1 is declared complete.

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
- missing optional icons,
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
10. SVG text portability policy: accessible `<text>` with pinned/embedded fonts versus conversion to paths.
11. Supported custom font formats and embedding policy.
12. Numeric implementation-wide logo limits for source bytes, dimensions, element count, nesting and filter regions; these are security caps distinct from per-asset display bounds.

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
build/payments/infrastructure/
├── diagram.d2
├── diagram.svg
└── manifest.json
```

The milestone validates the central concept only when the same model can subsequently generate at least one flow or sequence view without duplicating the system inventory.
