# FlowFrame

**FlowFrame is a lightweight diagram-as-code framework for creating consistent architecture, flow and sequence diagrams from structured semantic models.**

FlowFrame separates system facts, view intent and visual rendering. Humans or AI describe what a system contains and which view is required; a deterministic compiler validates those inputs, generates D2 and renders an SVG using a framework-owned design system.

> FlowFrame is currently at the specification stage. Start with the authoritative [product requirements](docs/prd.md), then use the [technical design](docs/technical-spec.md) and [implementation plan](docs/implementation-plan.md) for execution.

---

## Why FlowFrame?

- **Semantic source files** — system elements, boundaries, relations and scenarios are described in validated YAML.
- **Multiple views** — architecture, flow and sequence views can reuse the same System Model.
- **Consistent design** — colors, shapes, typography and connection styles belong to the framework.
- **Git-friendly** — source files and generated D2 are text-based and reviewable.
- **Deterministic core** — the same inputs and pinned toolchain produce the same D2.
- **Portable rendering** — ELK is the baseline layout engine; TALA is an optional extension.
- **AI-friendly, not AI-dependent** — AI may generate structured inputs but cannot bypass validation or control raw styling.
- **Offline by default** — validation and rendering must not require remote assets.

---

## How it works

```text
Natural language / source documentation
                 │
                 ▼
          optional AI adapter
                 │
                 ▼
   Project Configuration + System Model + View
                 │
                 ▼
       schema + semantic validation
                 │
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
        D2 validation and rendering
                 │
                 ▼
      SVG branding, accessibility and verification
                 │
                 ▼
      diagram.d2 + diagram.svg + manifest
```

The source of truth is:

```text
[flowframe.yaml + selected Theme/Brand Pack] + system-model.yaml + view.yaml
```

The generated artifacts are:

```text
diagram.d2
diagram.svg
manifest.json
```

Generated D2 is not intended for manual editing.

---

## MVP scope

FlowFrame v0.1 targets:

- `architecture/infrastructure`,
- `flow/integration-flow`,
- `sequence/sequence`,
- YAML input validated with JSON Schema,
- semantic validation and actionable diagnostics,
- deterministic D2 generation,
- offline SVG rendering with a pinned D2 CLI,
- ELK as the default layout engine,
- a centralized accessible light theme and one project-wide custom Theme/Brand Pack,
- golden-file and normalized SVG snapshot tests,
- one AI adapter producing System Model and View Specification files.

Deployment, system-context, network-security, data-flow, event-flow and dependency-flow views are planned after the initial MVP.

---

## System Model

The System Model contains reusable facts about the system. It distinguishes runtime elements from boundaries and keeps visual properties out of the source data.

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

relations:
  - id: gateway-calls-api
    source: app-gateway
    target: api
    semantic: request
    interaction: synchronous
    protocol: HTTPS
```

Vendor products use semantic kinds plus optional technology metadata. For example, Azure Service Bus remains a `queue`, and Azure Key Vault is a `secret-store`.

---

## View Specification

A view selects information from the System Model and defines its purpose without duplicating the system inventory.

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

display:
  boundaries: [environment, network]
  protocols: true
  technologies: true

layout:
  profile: hierarchical
  direction: right
```

Architecture, flow and sequence projections use separate normalized intermediate representations while retaining references to the original model IDs.

Project-wide colors, fonts, logo placement and footer formatting are selected once in `flowframe.yaml`. Individual views provide only presentation text; they cannot override the theme or logo.

---

## Layout engines

### ELK

ELK is the default and portable baseline. It is bundled with D2 and is the required engine for local and CI builds in v0.1.

### TALA

TALA is optional and planned after the baseline MVP. It requires a separate installation and a commercial license for commercial use. TALA-specific capabilities must remain isolated from the portable view contract.

FlowFrame never switches layout engines silently based on a subjective visual-quality assessment.

---

## Planned CLI

```text
flowframe validate --model MODEL [--view VIEW] [--config CONFIG]
flowframe validate --theme THEME_FILE
flowframe compile --model MODEL --view VIEW [--config CONFIG] --output diagram.d2
flowframe render --input diagram.d2 [--layout elk] [--config CONFIG] --output diagram.svg
flowframe build --model MODEL (--view VIEW | --all) [--config CONFIG] [--layout elk] --output-dir build/
flowframe review --model MODEL --view VIEW [--config CONFIG] [--mode MODE]
```

`build` produces the final branded diagram; `render` produces only its body and requires generated, policy-checked D2 plus matching theme configuration. `build --all` reads the config's explicit `views` list and writes to `build/<view-id>/`. Every processing command supports `--format text|json` and `--debug`.

`review` is added in Stage 4; syntax, semantic and policy modes work without AI. Source-conformance uses the optional adapter; visual review is post-MVP. `compare-layouts` is post-MVP because v0.1 requires only ELK. Its provisional contract is documented in the PRD and technical specification.

These commands describe the v0.1 contract and are not implemented yet.

---

## Design and accessibility

- Models and views cannot contain raw D2 styles or RGB/HEX colors.
- Custom colors are defined only in a validated project-wide `theme.yaml`.
- The project logo, title placement and source-footer template are configured globally in `flowframe.yaml`.
- Relation meaning must not rely on color alone.
- The baseline theme targets WCAG AA contrast.
- Diagrams must remain understandable in grayscale.
- Generic icons are local and licensed; remote icon URLs are disabled by default.
- Missing vendor icons fall back to semantic shapes.

---

## AI integration

AI is an optional adapter around the deterministic core. It may:

- extract or update a System Model,
- create a compatible View Specification,
- preserve stable IDs,
- report uncertain or missing information,
- use validation diagnostics to repair structured output.

AI must not generate authoritative D2 directly, invent raw styling or silently change system semantics during layout improvement.

---

## Development roadmap

1. Validate D2, ELK, optional TALA, offline assets and target SVG consumers.
2. Implement schemas, semantic validation, CLI and one infrastructure vertical slice.
3. Add the design system, accessibility checks and generic icons.
4. Add integration-flow and sequence projections.
5. Add the AI adapter and a fixed evaluation corpus.
6. Package the tool and provide CI integrations.
7. Add optional engines, diagram subtypes and vendor icon packs.

See [docs/prd.md](docs/prd.md) for requirements, acceptance criteria and risks; task sequencing is maintained in [the implementation plan](docs/implementation-plan.md).

---

## License

TBD
