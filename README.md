# Governance Rules Core

Production-tested governance rules for [CodeRadius](https://coderadius.ai).  
Each rule is a declarative **YAML file** containing a **Cypher query** that evaluates entities against an organizational standard using the live architecture graph.

> **These are not file-level lint rules.**  
> They are **topology-level checks** — cross-referencing service exposure, team ownership, dependency health, Docker registries, CI/CD configuration, and AI agent readiness **across the entire architecture graph**.

---

## Purpose

This repository is the **baseline rule set** for the CodeRadius governance engine. It provides ready-to-use policies that organizations can adopt, extend, or fork to enforce their own architectural standards.

The rules cover multiple governance domains:

| Domain | What it enforces |
|--------|-----------------|
| **CI/CD Compliance** | Pipeline existence, MR gates |
| **Code Quality** | TypeScript strict mode, compiler flag inheritance |
| **Developer Experience** | Makefile targets, DevContainers, AI agent onboarding |
| **Security & Supply Chain** | Deprecated/outdated dependencies, Renovate |
| **Service Catalog** | Backstage registration, team ownership |

---

## Principles

1. **Graph-native.** Rules query the CodeRadius architecture graph (Memgraph/Cypher), not individual files. This enables cross-entity policies impossible with traditional linters.
2. **Declarative.** Each rule is a self-contained YAML file. No imperative code, no plugins, no compilation step.
3. **Composable.** Organizations cherry-pick the rules they need. The engine can load multiple rule directories simultaneously.
4. **Extensible.** Write your own rules following the same YAML + Cypher contract. Place them in the `custom-rules/` directory (which is `.gitignore`d) to avoid committing proprietary, company-specific logic to this core repository.
5. **Severity-driven.** Each rule declares `error`, `warning`, or `info` — enabling progressive adoption without blocking deployments.
6. **Compliance-native.** Every rule returns **all in-scope entities** with a `pass` or `fail` status — enabling accurate compliance percentages and per-entity compliance tracking.

---

## Installation

Clone this repository into your CodeRadius workspace:

```bash
git clone https://github.com/coderadius-ai/governance-rules-core.git rules/governance
```

Rules are evaluated automatically during `radius policy check` and `radius dashboard` generation.

---

## Repository Structure

```
governance-rules-core/
├── README.md
├── custom-rules/          # Git-ignored folder for proprietary company rules
└── rules/                 # Standard, open-source baseline rules
```

The engine can load **all `.yaml` files** from multiple directories. Use `custom-rules/` for any company-specific policies (like custom CI toolkits or internal registry checks) to avoid committing them to this core repository.

### Numbering Convention

| Range | Category |
|-------|----------|
| `cr-1xx` | CI/CD Compliance |
| `cr-2xx` | Code Quality |
| `cr-3xx` | Developer Experience |
| `cr-4xx` | Security & Supply Chain |
| `cr-5xx` | Service Catalog & Ownership |

The first digit groups rules by category. While the current convention uses three digits (e.g., `cr-101`), this is just for readability; there is no hard limit on the number of rules per category. Gaps in numbering (e.g., jumping from `cr-101` to `cr-103`) are intentional and indicate deprecated rules.

---

## Rules Reference

### CI/CD Compliance (`cr-1xx`)

| ID | Name | Severity | Scope |
|----|------|----------|-------|
| `cr-101` | [CI/CD configuration required](#cr-101--cicd-configuration-required) | `error` | repository |
| `cr-103` | [MR pipeline gate active](#cr-103--mr-pipeline-gate-active) | `error` | repository |

### Code Quality (`cr-2xx`)

| ID | Name | Severity | Scope |
|----|------|----------|-------|
| `cr-201` | [TypeScript strict mode](#cr-201--typescript-strict-mode) | `error` | repository |

### Developer Experience (`cr-3xx`)

| ID | Name | Severity | Scope |
|----|------|----------|-------|
| `cr-301` | [Makefile mandatory targets](#cr-301--makefile-mandatory-targets) | `error` | repository |
| `cr-302` | [DevContainer configured](#cr-302--devcontainer-configured) | `info` | repository |
| `cr-303` | [AGENTS.md present](#cr-303--agentsmd-present) | `error` | repository |

### Security & Supply Chain (`cr-4xx`)

| ID | Name | Severity | Scope |
|----|------|----------|-------|
| `cr-402` | [Exposed services: deprecated dependencies](#cr-402--exposed-services-deprecated-dependencies) | `warning` | service |
| `cr-403` | [Exposed services: outdated dependencies](#cr-403--exposed-services-outdated-dependencies) | `warning` | service |
| `cr-404` | [Renovate configured](#cr-404--renovate-configured) | `warning` | repository |

### Service Catalog & Ownership (`cr-5xx`)

| ID | Name | Severity | Scope |
|----|------|----------|-------|
| `cr-501` | [Team ownership required](#cr-501--team-ownership-required) | `warning` | service |
| `cr-502` | [Backstage catalog registered](#cr-502--backstage-catalog-registered) | `error` | repository |

---

## Rule Details

### `cr-101` — CI/CD Configuration Required

All repositories must have a CI/CD pipeline configuration file. Supported: `.gitlab-ci.yml`, `.github/workflows/*.yml`.

- **Severity:** `error` · **Scope:** `repository`
- **Tags:** `ci-cd`, `compliance`

### `cr-103` — MR Pipeline Gate Active

Merge Request pipelines ensure that every PR - whether produced by a human or an AI agent - passes automated tests before merge. Without this control, AI agents can merge untested code autonomously.

- **Severity:** `error` · **Scope:** `repository`
- **Tags:** `ci-cd`, `compliance`, `ai-readiness`

### `cr-201` — TypeScript Strict Mode

Every TypeScript project must have `strict: true`, `noFallthroughCasesInSwitch: true`, and `noUncheckedIndexedAccess: true` in `tsconfig.json`.

- **Severity:** `error` · **Scope:** `repository`
- **Tags:** `code-quality`, `typescript`
- **Details:** Uses `resolvedStrict` / `resolvedNoFallthrough` / `resolvedNoUnchecked` properties which account for **tsconfig inheritance chains** (`extends`). Covers both repository-level and service-level tsconfig files in monorepos.

### `cr-301` — Makefile Mandatory Targets

Every repository must define `setup`, `test`, and `run` Makefile targets. These targets standardize developer workflows and enable consistent CI/CD pipelines.

- **Severity:** `error` · **Scope:** `repository`
- **Tags:** `developer-experience`, `makefile`
- **Detail:** Returns a checklist of missing targets with pass/fail status per target via `structuredDetail`.

### `cr-302` — DevContainer Configured

A `devcontainer.json` ensures that AI agents and human developers operate in the same reproducible environment. Eliminates "works on my machine" issues.

- **Severity:** `info` · **Scope:** `repository`
- **Tags:** `developer-experience`, `ai-readiness`, `environment`

### `cr-303` — AGENTS.md Present

`AGENTS.md` is the file that AI agents read before operating on any repository file. It contains code conventions, local architecture, test commands, and service-specific operational instructions.

- **Severity:** `error` · **Scope:** `repository`
- **Tags:** `developer-experience`, `ai-readiness`, `documentation`
- **Why it matters:** Without `AGENTS.md`, every AI agent session starts without context - increasing the risk of incorrect architectural decisions and silent regressions.

### `cr-402` — Exposed Services: Deprecated Dependencies

Services that expose public API endpoints must not depend on packages with explicitly deprecated releases. This is a **topology-aware** policy — it cross-references service exposure (via `APIEndpoint`) with package health (via `Release.deprecated=true`).

- **Severity:** `warning` · **Scope:** `service`
- **Tags:** `security`, `dependencies`, `topology`
- **Why topology matters:** This check is impossible with file-level linting. It requires knowing (a) which services expose HTTP APIs, and (b) which of their transitive dependencies have deprecated releases - two facts that live in completely different parts of the codebase.

### `cr-403` — Exposed Services: Outdated Dependencies

Services that expose public API endpoints have a higher security and reliability responsibility. If any of their dependencies have a known newer version available, they should be updated promptly.

- **Severity:** `warning` · **Scope:** `service`
- **Tags:** `security`, `dependencies`, `topology`

### `cr-404` — Renovate Configured

Renovate ensures dependencies are automatically updated with dedicated PRs. Without it, outdated libraries become a silent security risk.

- **Severity:** `warning` · **Scope:** `repository`
- **Tags:** `security`, `dependencies`, `automation`
- **Checks for:** `renovate.json`, `renovate.json5`, `.renovaterc`, `.renovaterc.json`, `.github/renovate.json`

### `cr-501` — Team Ownership Required

Every Service in the architecture graph must be owned by at least one Team. Unowned services lack a clear on-call contact and complicate incident response.

- **Severity:** `warning` · **Scope:** `service`
- **Tags:** `service-catalog`, `ownership`

### `cr-502` — Backstage Catalog Registered

Every repository must register in the Backstage Software Catalog via a `catalog-info.yaml` file. Without it, AI agents and platform tools have no visibility into service topology.

- **Severity:** `error` · **Scope:** `repository`
- **Tags:** `service-catalog`, `ai-readiness`, `backstage`

---

## How It Works

```
┌──────────────┐     ┌──────────────────┐     ┌───────────────────┐
│  YAML Rule   │────>│  CodeRadius CLI   │────>│  Architecture     │
│  (this repo) │     │  radius policy    │     │  Graph (Memgraph) │
└──────────────┘     └──────────────────┘     └───────────────────┘
                              │
                              ▼
                     ┌──────────────────┐
                     │  Evaluations     │
                     │  (pass + fail)   │
                     └──────────────────┘
```

1. **Ingest** — `radius ingest` scans your repositories and builds the architecture graph in Memgraph.
2. **Load** — The engine discovers all `.yaml` files under your configured rule directories (e.g., `rules/` and `custom-rules/`).
3. **Evaluate** — Each rule's Cypher `query` runs against the graph. Every returned row is an **evaluation** with a `pass` or `fail` status.
4. **Report** — Evaluations are surfaced in the CLI output, the dashboard compliance tab, or CI pipeline checks.

Each row returned = one evaluation. `status = 'pass'` = compliant, `status = 'fail'` = violation.

---

## Writing Custom Rules

When writing company-specific policies, **avoid using numeric prefixes** (e.g. `cr-900`) to prevent ID collisions across teams. Instead, use a **semantic slug with a namespace**.

Each rule is a YAML file with the following structure:

```yaml
id: acme/require-readme
name: My custom rule
description: >
  Human-readable explanation of what this rule checks
  and why it matters.
severity: error          # error | warning | info
scope: repository        # repository | service | package
tags:
  - custom
  - my-domain

query: |
  MATCH (r:Repository)
  OPTIONAL MATCH (r)-[:HAS_CONFIG]->(sf:StructuralFile)
    WHERE sf.path = 'README.md'
  WITH r, sf
  RETURN
    r.id         AS entityId,
    r.name       AS entityName,
    'repository' AS entityType,
    CASE WHEN sf IS NULL THEN 'fail' ELSE 'pass' END AS status,
    CASE WHEN sf IS NULL THEN 'README.md missing' ELSE '' END AS detail
```

### Query Contract

Every rule's Cypher query **must** return rows with these columns:

| Column | Required | Description |
|--------|----------|-------------|
| `entityId` | ✅ | Unique ID of the evaluated entity |
| `entityName` | ✅ | Display name |
| `entityType` | ✅ | Must match `scope`: `repository`, `service`, or `package` |
| `status` | ✅ | `'pass'` (compliant) or `'fail'` (violation) |
| `detail` | ✅ | Human-readable explanation (required for `fail`, may be empty for `pass`) |
| `structuredDetail` | ❌ | Optional structured payload for rich dashboard rendering |

### YAML Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | ✅ | Unique rule identifier (e.g. `cr-301-makefile-targets` or `acme/require-readme`) |
| `name` | ✅ | Short human-readable name |
| `description` | ✅ | Full explanation of what the rule checks and why |
| `severity` | ✅ | `error`, `warning`, or `info` |
| `scope` | ✅ | `repository`, `service`, or `package` |
| `tags` | ❌ | Categorization labels for filtering |
| `failFast` | ❌ | If `true`, stop evaluation on first failure (default: `false`) |
| `query` | ✅ | Cypher query that returns evaluations (pass + fail) |

### Tips

- **Test in Memgraph Lab first.** Write and debug your Cypher query interactively before wrapping it in YAML.
- **Return all in-scope entities.** Use `CASE WHEN ... THEN 'fail' ELSE 'pass' END AS status` — do not filter out compliant entities.
- **Use `structuredDetail` for multi-item checks.** Return a `{ checks, found }` map for rich checklist rendering.
- **Include the `found` array.** If your rule checks for expected items, also return the actual items found. This enables fuzzy matching in the UI.

---

## Examples

### Example 1: Check that all repos have a README

```yaml
id: cr-301-readme-exists
name: README.md required
description: >
  Every repository must have a README.md file to provide basic documentation
  for developers and AI agents.
severity: warning
scope: repository
tags: [documentation, developer-experience]

query: |
  MATCH (r:Repository)
  OPTIONAL MATCH (r)-[:HAS_CONFIG]->(sf:StructuralFile)
    WHERE sf.path = 'README.md'
  WITH r, sf
  RETURN
    r.id         AS entityId,
    r.name       AS entityName,
    'repository' AS entityType,
    CASE WHEN sf IS NULL THEN 'fail' ELSE 'pass' END AS status,
    CASE WHEN sf IS NULL THEN 'README.md missing' ELSE '' END AS detail
```

### Example 2: Services using more than N external dependencies

```yaml
id: cr-401-dependency-sprawl
name: Dependency sprawl check
description: >
  Services with more than 50 external dependencies should be reviewed
  for potential splitting or dependency consolidation.
severity: info
scope: service
tags: [architecture, dependencies]

query: |
  MATCH (s:Service)
  OPTIONAL MATCH (s)-[:DEPENDS_ON]->(p:Package)
  WHERE p.isInternal = false
  WITH s, count(p) AS depCount
  RETURN
    s.id      AS entityId,
    s.name    AS entityName,
    'service' AS entityType,
    CASE WHEN depCount > 50 THEN 'fail' ELSE 'pass' END AS status,
    CASE WHEN depCount > 50
      THEN 'Service has ' + toString(depCount) + ' external dependencies (threshold: 50)'
      ELSE '' END AS detail
```

### Example 3: Topology-aware — unprotected database access

```yaml
id: cr-401-direct-db-from-api
name: API services must not access database directly
description: >
  Services exposing public APIs should access data through a dedicated
  data service, not directly connecting to the database.
severity: warning
scope: service
tags: [architecture, topology, security]

query: |
  MATCH (s:Service)-[:EXPOSES_API]->(:APIInterface)
  OPTIONAL MATCH (s)-[:CONNECTS_TO]->(db:Database)
  RETURN
    s.id      AS entityId,
    s.name    AS entityName,
    'service' AS entityType,
    CASE WHEN db IS NOT NULL THEN 'fail' ELSE 'pass' END AS status,
    CASE WHEN db IS NOT NULL
      THEN 'Public API service directly connects to ' + db.name + ' - consider using a data service layer'
      ELSE '' END AS detail
```

---

## Documentation

- [Governance Engine Docs](https://coderadius.ai/docs/governance) — Full documentation for the governance engine
- [CodeRadius CLI](https://github.com/coderadius-ai/cli) — The CLI that evaluates these rules
- [Graph Taxonomy](https://coderadius.ai/docs/architecture/graph-taxonomy) — Node types and relationships available for rule queries

## License

MIT