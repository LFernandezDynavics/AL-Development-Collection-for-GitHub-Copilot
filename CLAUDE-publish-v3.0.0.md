# Tarea 9: Publicar ALDC Core v1.1 como extensión v3.0.0

## Contexto

La extensión actual en el VS Code Marketplace es ALDC (AL Development Collection).
La versión actual del marketplace es 2.11.x.
ALDC Core v1.1 es una reorganización mayor: skills-based modularization, 
roles corregidos (architect/spec.create), implement-subagent restaurado.
Se publica como v3.0.0 (breaking changes).

## 9A. Actualizar package.json

```json
{
  "name": "aldc-al-development-collection",
  "displayName": "ALDC - AL Development Collection for GitHub Copilot",
  "description": "Framework for building Business Central AL extensions with GitHub Copilot agents. Spec-first, architecture-driven, TDD-orchestrated development with human-in-the-loop gates.",
  "version": "3.0.0",
  "publisher": "JavierArmesto",
  "license": "MIT",
  "engines": {
    "vscode": "^1.95.0"
  },
  "categories": [
    "AI",
    "Programming Languages",
    "Other"
  ],
  "keywords": [
    "business-central",
    "al",
    "dynamics-365",
    "copilot",
    "github-copilot",
    "agents",
    "tdd",
    "framework",
    "aldc"
  ],
  "repository": {
    "type": "git",
    "url": "https://github.com/javiarmesto/ALDC-AL-Development-Collection-for-GitHub-Copilot"
  },
  "icon": "docs/assets/aldc-icon.png",
  "galleryBanner": {
    "color": "#1E1E1E",
    "theme": "dark"
  }
}
```

Mantener el resto de campos existentes (activationEvents, contributes, etc.).
Solo actualizar version, description y keywords.

## 9B. Actualizar README.md para el marketplace

El README es lo que se muestra en la página del marketplace. Debe reflejar v3.0.0:

### Estructura del README:

```markdown
# ALDC - AL Development Collection for GitHub Copilot

> A practical framework for building Business Central AL extensions with GitHub Copilot agents.  
> From vibe coding to controlled engineering.

![ALDC Core v1.1 Compliant](badge-url)

## What is ALDC?

ALDC (AL Development Collection) is a framework that transforms how you develop 
Business Central extensions with GitHub Copilot. Instead of ad-hoc code generation,
ALDC provides structured, contract-driven development with human-in-the-loop gates.

## Key Features

**4 Public Agents** — Specialized roles for every development phase
- `@al-architect` — Solution Architect: designs solutions, information flows, technical decisions
- `@al-developer` — Developer: implements, debugs, quick adjustments
- `@al-conductor` — Conductor: orchestrates TDD implementation with subagents
- `@al-presales` — Pre-sales: estimation and scoping

**3 Internal Subagents** — Autonomous specialists within the conductor
- `al-planning-subagent` — Research and context gathering
- `al-implement-subagent` — TDD-only implementation (tests FIRST, code SECOND)
- `al-review-subagent` — Code review against spec + architecture

**11 Composable Skills** — Domain knowledge loaded on demand
- Required: api, copilot, debug, performance, events, permissions, testing
- Recommended: migrate, pages, translate, estimation

**6 Workflows** — Automated processes
- `al-spec.create`, `al-build`, `al-pr-prepare`, `al-context.create`, `al-memory.create`, `al-performance`

**Contracts per Requirement** — Structured documentation
- `architecture.md` — Solution design (from architect)
- `spec.md` — Technical blueprint (from spec.create)
- `test-plan.md` — Test strategy
- `memory.md` — Global context across sessions

## How It Works

### Development Flow

```
LOW complexity:
  al-spec.create → @al-developer

MEDIUM/HIGH complexity:
  @al-architect → al-spec.create → @al-conductor
```

The architect can decompose complex requirements into multiple specs,
each implemented independently by the conductor.

### TDD Orchestration

The conductor enforces Test-Driven Development:
1. Planning subagent researches context
2. Implement subagent creates tests FIRST (RED)
3. Implement subagent writes code to pass tests (GREEN)
4. Review subagent validates against spec + architecture
5. Human approves each phase (HITL gate)
6. Phase complete → next phase

## Installation

Install from VS Code Marketplace or:
```bash
code --install-extension JavierArmesto.aldc-al-development-collection
```

After installation, the framework copies to your project's `.github/` directory.

## Quick Start

1. Install the extension
2. Open your AL project
3. Run: `@workspace use al-spec.create` with your requirement
4. Follow the guided flow

See [QUICKSTART.md](docs/framework/QUICKSTART.md) for the 30-minute onboarding guide.

## What's New in v3.0.0

- **Skills-based modularization**: 11 composable skills replace 7 specialized agents
- **Restored TDD enforcement**: `al-implement-subagent` with hardcoded TDD cycle
- **Corrected agent roles**: architect = Solution Architect, spec.create = technical blueprint
- **Contracts per requirement**: subdirectory structure `.github/plans/{req_name}/`
- **Skills evidencing**: agents declare which skills they load and apply
- **HITL gates enforced**: mandatory stops at plan approval, each phase, and completion

### Breaking Changes from v2.x

- Agents reduced from 11 to 4 public + 3 internal subagents
- Agent references changed (e.g., `al-architect-mode` → `@al-architect`)
- Plans directory structure changed to per-requirement subdirectories
- `al-implement-subagent` restored (was merged into `al-developer` in early v1.1)

See [CHANGELOG.md](CHANGELOG.md) for full details.

## Framework Documentation

- [Core Specification v1.1](docs/framework/ALDC-Core-Spec-v1.1.md)
- [Manifesto](docs/framework/ALDC-Manifesto.md)
- [Quickstart](docs/framework/QUICKSTART.md)
- [Roadmap 2026](docs/framework/ROADMAP-2026.md)
- [Governance](docs/framework/ALDC-Governance.md)

## ALDC Core Compliant

Validate your project:
```bash
node tools/aldc-validate/index.js --config aldc.yaml
```

## Author

**Javier Armesto González**  
Microsoft MVP (Business Central & Azure AI Services)  
Head of R&D & AI at VS Sistemas

## License

MIT
```

## 9C. Actualizar CHANGELOG.md

Añadir al principio:

```markdown
## [3.0.0] - 2026-03-XX

### ALDC Core v1.1 — Skills-based modularization with preserved orchestration

**Breaking Changes:**
- Agents reorganized: 11 → 4 public + 3 internal subagents + 11 skills
- Agent naming: `al-architect-mode` → `@al-architect`, etc.
- Plans structure: flat → per-requirement subdirectories
- al-implement-subagent restored as TDD-only subagent
- Agent roles corrected: architect = Solution Architect, spec.create = technical blueprint

**Added:**
- 11 composable skills (api, copilot, debug, performance, events, permissions, testing, migrate, pages, translate, estimation)
- al-implement-subagent with full AL development capabilities + TDD enforcement
- Skills evidencing: agents declare loaded skills and applied patterns
- Per-requirement directory structure in .github/plans/
- Phase 1 (Planning) completion document
- Test infrastructure verification (Library Assert, Any, ID range checks)
- Solution architecture capabilities: information flow diagrams, requirement decomposition

**Fixed:**
- TDD enforcement lost during agent consolidation (0 tests → enforced TDD)
- Review subagent not invoked by conductor
- HITL gates skipped between phases
- Phase completion documents not generated
- test-plan.md not created as part of requirement set
- memory.md not updated at plan completion
- Test compilation errors (Assert naming, Any dependency, ID range)

**Changed:**
- Narrative: "simplification" → "modularization with preserved orchestration"
- Architect role: from lightweight agent to full Solution Architect
- Spec.create role: from functional spec to detailed technical blueprint
- Flow: @al-architect → al-spec.create → @al-conductor (MEDIUM/HIGH)
```

## 9D. Crear/actualizar aldc.yaml

```yaml
core:
  version: "1.1.0"

metadata:
  updated: "2026-03-XX"
```

## 9E. Publicar

```bash
# Desde la raíz del repo
# 1. Verificar que todo compila
node tools/aldc-validate/index.js --config aldc.yaml

# 2. Package
vsce package

# 3. Publicar
vsce publish
```

## 9F. Post-publicación

1. Crear GitHub Release v3.0.0 con tag
2. Actualizar GitHub repo description
3. LinkedIn post anunciando v3.0.0
4. Actualizar Substack (TechSphere Dynamics) con artículo sobre v3.0.0
