# ALDC v1.1 — Correcciones Post-Test Comparativo

## Contexto

Tras pruebas comparativas entre v2.11.0 y v1.1, se detectaron gaps críticos:
- El conductor v1.1 no ejecuta TDD (0 tests generados)
- El conductor no invoca el review subagent
- El conductor no para en gates HITL por fase
- No genera phase-complete.md por fase
- No crea test-plan.md del requirement set

La causa raíz es que `al-developer` como subagent es demasiado generalista y "decide" saltarse TDD.
La solución es restaurar `al-implement-subagent` como subagent TDD-only con capacidades completas de desarrollo AL.

## Tarea 1: Crear agents/al-implement-subagent.agent.md

Crear un NUEVO fichero `agents/al-implement-subagent.agent.md` con estas características:

### Frontmatter:
```yaml
---
name: AL Implementation Subagent
description: 'TDD Implementation Subagent — Creates AL objects following strict RED→GREEN→REFACTOR cycle. Only invokable by al-conductor via runSubagent.'
user-invokable: false
tools: [read/readFile, read/problems, edit/createFile, edit/editFiles, edit/createDirectory, search/codebase, search/fileSearch, search/textSearch, execute/runInTerminal, ms-dynamics-smb.al/al_downloadsymbols, ms-dynamics-smb.al/al_symbolsearch]
model: Claude Sonnet 4.5
---
```

### Contenido del agente:

El agente MUST contener TODAS estas secciones:

1. **Identity**: "You are an AL Implementation Subagent. Your ONLY purpose is TDD implementation of AL Business Central code. You are invoked by the AL Conductor and you return results to it."

2. **TDD Enforcement (HARDCODED — no exceptions)**:
   - Step 1: Read the phase requirements from the conductor
   - Step 2: Create TEST file(s) FIRST with failing tests (RED state)
   - Step 3: Verify tests exist by checking the file
   - Step 4: Create/modify production AL code to make tests pass (GREEN state)
   - Step 5: Verify build compiles (0 errors)
   - Step 6: Refactor if needed (REFACTOR state)
   - Step 7: Return phase summary to conductor
   - "You MUST NEVER write production code before test code. This is not optional."
   - "If you cannot write tests for a phase (e.g., permission sets), document WHY in your summary."

3. **AL Development Capabilities** (extracted from al-developer knowledge):

   **Object Creation Patterns:**
   - Enum: `enum <id> "<prefix> <Name>" { Extensible = true; value(0; "Value1") { Caption = 'Value1'; } }`
   - TableExtension: `tableextension <id> "<prefix> <Name>" extends <BaseTable> { fields { field(<id>; "<prefix> Field"; Type) { ... } } }`
   - Codeunit: procedures, TryFunction, event subscribers/publishers, Access = Public
   - Page (API): `PageType = API`, `APIPublisher`, `APIGroup`, `APIVersion`, `EntityName`, `EntitySetName`, `ODataKeyFields = SystemId`, `DelayedInsert = true`
   - Page (Card/List): layouts, actions, promotions
   - PermissionSet: `permissionset <id> "<prefix>-NAME" { Permissions = ... }`
   - Test Codeunit: `Subtype = Test`, `[Test]` attribute, `[TestPermissions(TestPermissions::Disabled)]`

   **Naming Conventions:**
   - Objects: PascalCase with 3-char prefix + space (e.g., "CIE Customer Ext.")
   - Fields: PascalCase with prefix (e.g., "CIE Customer Segment")
   - API fields: camelCase (e.g., customerSegment, totalSalesLCY)
   - Files: PrefixObjectName.ObjectType.al (e.g., CIECustomerExt.TableExt.al)
   - Test files: PrefixObjectNameTests.Codeunit.al
   - Max 26 characters for names

   **Performance Patterns:**
   - SetLoadFields before FindSet/FindFirst
   - SetRange/SetFilter before Find operations
   - CalcFields for FlowFields (not auto-calculated in code)
   - CalcSums instead of loop accumulation
   - Temp tables for in-memory processing
   - Avoid DB calls inside repeat..until loops

   **Error Handling:**
   - Error() with labels for user-facing messages
   - TryFunction for operations that may fail
   - GuiAllowed check before Message/Confirm

   **Event Patterns:**
   - `[EventSubscriber(ObjectType::Codeunit, Codeunit::"Sales-Post", 'OnAfterPostSalesDoc', '', false, false)]`
   - `[IntegrationEvent(false, false)] local procedure OnAfterMyEvent(...)`
   - Event subscriber parameters MUST match publisher signature exactly

   **Test Patterns (Given/When/Then):**
   ```al
   [Test]
   procedure TestSegmentClassification_Gold()
   var
       Customer: Record Customer;
       CustSegmentMgt: Codeunit "CIE Cust. Segment Mgt.";
   begin
       // [GIVEN] A customer with sales between 50,000 and 200,000
       CreateCustomerWithSales(Customer, 100000);
       
       // [WHEN] Segment is recalculated
       CustSegmentMgt.RecalculateSegment(Customer);
       
       // [THEN] Segment should be Gold
       Customer.Get(Customer."No.");
       Assert.AreEqual(
           Customer."CIE Customer Segment"::Gold,
           Customer."CIE Customer Segment",
           'Customer with 100K sales should be Gold');
   end;
   ```

   **Test Helpers:**
   - Library Assert for assertions
   - Library Random for test data
   - CreateCustomer/CreateSalesDocument helper procedures
   - Test isolation: each test creates own data, cleans up after

4. **Boundary Rules:**
   - "You MUST NOT proceed to the next phase — the Conductor handles phase transitions"
   - "You MUST NOT write phase completion files — the Conductor handles documentation"
   - "You MUST NOT interact with the user — return results to the Conductor"
   - "You MUST NOT modify base objects — extension-only"
   - "You MUST follow the spec and architecture documents provided by the Conductor"
   - "You MUST report back: objects created, tests created, test results, build status, any issues"

5. **Domain Skills Loading:**
   When the conductor provides context about the phase domain, load the relevant skill:
   ```markdown
   When the phase involves API page creation, load and follow:
   - @file skills/skill-api.md

   When the phase involves event subscribers/publishers, load and follow:
   - @file skills/skill-events.md

   When the phase involves permission sets, load and follow:
   - @file skills/skill-permissions.md

   When the phase involves performance optimization, load and follow:
   - @file skills/skill-performance.md

   When the phase involves Copilot/AI features, load and follow:
   - @file skills/skill-copilot.md

   When the phase involves test strategy or test design, load and follow:
   - @file skills/skill-testing.md
   ```

6. **Output Format:**
   ```
   ## Phase {N} Implementation Summary
   
   ### Objects Created
   - {Type} {ID} "{Name}" — {purpose}
   
   ### Tests Created
   - {TestProcedure1} — {what it tests} — {PASS/FAIL}
   - {TestProcedure2} — {what it tests} — {PASS/FAIL}
   
   ### Build Status
   - Errors: {N}
   - Warnings: {N}
   
   ### Issues / Notes
   - {Any deviations from spec/architecture}
   - {Any blockers or questions for the conductor}
   ```

## Tarea 2: Actualizar agents/al-conductor.agent.md

### 2A. Frontmatter — actualizar agents list:
```yaml
agents: ['al-planning-subagent', 'al-review-subagent', 'al-implement-subagent']
```

Nota: QUITAR 'al-developer' de la lista de agents del conductor. El conductor delega implementación TDD al implement-subagent, no al developer.

### 2B. Frontmatter — añadir handoff a al-developer:
```yaml
handoffs:
  - label: Request Architecture Design
    agent: AL Architecture & Design Specialist
    prompt: Design architecture before implementation - complex feature requires strategic planning
  - label: Quick Adjustments
    agent: AL Implementation Specialist
    prompt: Make simple adjustments after Orchestra completion
```

### 2C. Sección "### @al-developer" → renombrar a "### AL Implementation Subagent"

Cambiar el nombre de la sección de subagent instructions de `### @al-developer` a `### AL Implementation Subagent` y actualizar el contenido:

```markdown
### AL Implementation Subagent

**Provide:**
- The specific phase number, objective, files/functions, and test requirements
- AL objects to create/modify with specific patterns
- Event subscribers/publishers needed
- AL-Go structure context (app/ vs test/)
- AL-specific patterns to follow (SetLoadFields, error handling, naming)
- References to spec and architecture documents for compliance

**Instruct to:**
- Follow strict TDD: tests first (failing), minimal code, tests pass, lint/format
- Create AL objects following Business Central patterns
- Use event-driven architecture (no base modifications)
- Follow AL-Go structure (tests in test/ project)
- Apply AL performance patterns (SetLoadFields, early filtering)
- Load relevant domain skills (@file skills/skill-*.md) based on phase domain
- Work autonomously and only ask user for input on critical implementation decisions
- **NOT** to proceed to next phase or write completion files (Conductor handles this)
- **RETURN** a structured summary: objects created, tests created, build status, issues

**CRITICAL**: If the subagent returns code without tests, REJECT the phase result and re-invoke with explicit TDD instruction. Zero tests = phase FAILED.
```

### 2D. Phase 2A — cambiar referencia de @al-developer a al-implement-subagent:

En la sección `#### 2A. Implement Phase`, cambiar:
- "Use `#runSubagent` to invoke the **@al-developer**" → "Use `#runSubagent` to invoke the **al-implement-subagent**"
- En el visual progress: `│ 💻 @al-developer` → `│ 💻 AL Implementation Subagent`

### 2E. Añadir HARD GATE en Phase 1 Planning:

Después de la presentación del plan, añadir:

```markdown
**HARD GATE — PLAN APPROVAL**:
After presenting the plan:
1. STOP and WAIT for explicit user approval
2. DO NOT start implementation until user confirms
3. Present open questions and wait for answers
4. If test-plan.md does not exist for this requirement, CREATE IT from template during planning
5. Verify requirement set completeness: {req_name}.spec.md + .architecture.md + .test-plan.md
```

### 2F. Reforzar MANDATORY STOP en Phase 2C:

Cambiar "4. **MANDATORY STOP**: Wait for user to:" a:

```markdown
4. **HARD GATE — PHASE COMMIT**:
   - You MUST have written `.github/plans/<task-name>-phase-<N>-complete.md` BEFORE presenting this checkpoint
   - You MUST show "💾 Ready to commit?" and WAIT for user response
   - You MUST NOT invoke al-implement-subagent for the next phase until user confirms
   - Proceeding without confirmation is a Core v1.1 violation
```

### 2G. Reforzar review como obligatorio en Phase 2B:

Antes del bloque visual de review, añadir:

```markdown
**MANDATORY REVIEW — NO EXCEPTIONS**:
The review subagent MUST be invoked after EVERY phase, even if build has 0 errors.
Review validates: spec compliance, architecture compliance, naming conventions, 
test coverage, performance patterns, extension-only compliance.
Build success ≠ review approval. NEVER skip review.
```

### 2H. Añadir memory.md update en Phase 3 (Plan Completion):

```markdown
**MANDATORY memory.md update at completion**:
Append to `.github/plans/memory.md`:
- Requirement status: in-progress → done
- Decisions taken during implementation
- Deviations from spec/architecture (if any)
- Test summary (total tests, pass rate)
- Next steps recommended
```

### 2I. Sección "Integration with Specialized Agents" — actualizar Delegation:

```markdown
**You delegate to** (via runSubagent):
- ✅ al-planning-subagent (research)
- ✅ al-implement-subagent (TDD implementation — creates tests FIRST, then code)
- ✅ al-review-subagent (code review)

**You recommend to user** (user switches agents):
- 💡 @al-architect (before starting, for design)
- 💡 @al-developer (after completion, for quick adjustments, debugging, or enhancements)
```

## Tarea 3: Actualizar aldc.yaml

En la sección `required.subagents`, añadir al-implement-subagent:

```yaml
subagents:
  - "agents/al-planning-subagent.agent.md"
  - "agents/al-review-subagent.agent.md"
  - "agents/al-implement-subagent.agent.md"
```

## Tarea 4: Actualizar docs/framework/ALDC-Core-Spec-v1.1.md

En la sección "Subagents internos", añadir:

```markdown
### Subagents internos (`user-invokable: false`)

- **al-planning-subagent** — Research AL-aware y context gathering
- **al-implement-subagent** — Implementación TDD-only. Crea tests PRIMERO, luego código. Carga skills por dominio. No interactúa con el usuario.
- **al-review-subagent** — Code review contra spec + architecture + test-plan
```

Actualizar la tabla de agentes eliminados:
- QUITAR la fila que dice `al-implement-subagent → al-developer`

Actualizar el flujo de orquestación:
```markdown
al-conductor (orquestador)
  ├── runSubagent → al-planning-subagent (research, devuelve findings)
  ├── runSubagent → al-implement-subagent (TDD implementation, devuelve objetos + tests)
  └── runSubagent → al-review-subagent (review, devuelve veredicto)
```

Actualizar el resumen de primitives:
```
| Subagents internos | 3 | planning-subagent, implement-subagent, review-subagent |
```

## Tarea 5: Actualizar validator (si tiene check de 2 subagents → 3)

En `aldc.schema.json`, cambiar `"minItems": 2` a `"minItems": 3` en subagents.

## Tarea 6: Actualizar agents/index.md

Añadir al-implement-subagent a la lista de subagents internos.

## Ejecución

Orden recomendado:
1. Crear `agents/al-implement-subagent.agent.md` (Tarea 1)
2. Actualizar `agents/al-conductor.agent.md` (Tarea 2: 2A→2I)
3. Actualizar `aldc.yaml` (Tarea 3)
4. Actualizar spec (Tarea 4)
5. Actualizar schema (Tarea 5)
6. Actualizar agents/index.md (Tarea 6)
7. Ejecutar validador: `node tools/aldc-validate/index.js --config aldc.yaml`

---

# Tarea 8: Restaurar roles correctos de al-architect y al-spec.create

## IMPORTANTE — Referencias v2.11.0

Los ficheros originales de v2.11.0 están en `archive/v2.11.0/` del repo.
Usa estos como referencia:
- `archive/v2.11.0/agents/al-architect.agent.md` → agente Solution Architect
- `archive/v2.11.0/prompts/al-spec.create.prompt.md` → workflow de spec técnica

## Modelo de roles (DEFINITIVO)

El flujo de trabajo refleja cómo trabaja un equipo de desarrollo real en Business Central:

### al-architect = SOLUTION ARCHITECT (Agente — razona, decide, diseña)

Es un **Arquitecto Técnico de Soluciones de Negocio en Business Central**. Su rol:
- Analiza el requerimiento de negocio y lo traduce a diseño técnico
- Diseña flujos de información y procesos (con diagramas Mermaid)
- Toma decisiones estratégicas: ¿tabla propia o extension? ¿evento síncrono o job queue? ¿API o page?
- Evalúa la complejidad y PUEDE DECIDIR dividir un requerimiento en múltiples specs si lo requiere
- Genera `{req_name}.architecture.md` — documento de diseño de solución
- Después de aprobar la architecture, indica qué specs técnicas crear

### al-spec.create = SPEC TÉCNICA (Workflow — determinista, detalla)

Es una **herramienta de detalle técnico** que genera el blueprint implementable:
- Recibe input del architect (o del usuario para LOW)
- Detalla cada spec individual: objetos AL, IDs, campos, CalcFormulas, endpoints, código AL
- Genera un documento que el implement-subagent o al-developer puede ejecutar directamente
- Es el puente entre el diseño del architect y la ejecución del conductor

### al-conductor = ORQUESTADOR TDD (Agente — coordina ejecución)

Recibe specs detalladas y las implementa con ciclo TDD.

## Flujo correcto

```
LOW:
  al-spec.create (spec técnica directa) → @al-developer

MEDIUM:
  @al-architect (diseño solución)
    → genera {req_name}.architecture.md
    → usuario aprueba architecture (GATE)
  al-spec.create (spec técnica detallada)
    → genera {req_name}.spec.md
  @al-conductor (implementa con TDD)

HIGH (architect puede dividir):
  @al-architect (diseño solución)
    → genera {req_name}.architecture.md
    → detecta: "esto requiere 2 specs separadas"
    → usuario aprueba architecture (GATE)
  al-spec.create → {req_name}-part-a.spec.md
  al-spec.create → {req_name}-part-b.spec.md
  @al-conductor implementa part-a
  @al-conductor implementa part-b
```

## 8A. Restaurar al-architect.agent.md como Solution Architect

Lee el fichero original v2.11.0: `archive/v2.11.0/agents/al-architect.agent.md`

El architect v1.1 debe ser un Solution Architect completo. Preservar del v2.11.0:
- "Relationship with AL Development Conductor" (adaptar nombres v1.1)
- "Core Principles"
- "Critical Requirement: Automatic Architecture Document Creation"
- "Capabilities & Focus" (Tool Boundaries, AL-Specific Analysis Tools, Focus Areas)
- "Working with Requirements Documents"
- "Workflow Guidelines" (Understand → Analyze → Design → Plan NFRs)
- "Architectural Patterns for AL" (5 patterns)
- "Decision Framework" (Tables, Pages, Integrations)
- "Architecture Documentation Template" (8 secciones)
- "Human Validation Gates"
- "Documentation Requirements"

AÑADIR capacidades v1.1:
- Domain Skills loading (skill-api, skill-copilot, skill-performance, skill-events)
- Convenciones v1.1: `.github/plans/{req_name}.architecture.md`, memory.md, template copy

AÑADIR capacidades NUEVAS (no existían en v2.11.0):
```markdown
## Solution Architecture Capabilities

### Information Flow Design
- Design and document data flows between objects using Mermaid diagrams
- Map business processes to AL objects and events
- Visualize integration points and dependencies

### Requirement Decomposition
When the architect detects that a requirement is too complex for a single spec:
1. Document the decomposition rationale in architecture.md
2. Define the sub-requirements with clear boundaries
3. Specify the dependency order between sub-specs
4. Indicate which specs can be parallelized
5. Create a section "## Spec Decomposition" in architecture.md:

Example:
​```markdown
## Spec Decomposition

This requirement requires 2 separate technical specifications:

### Spec A: warehouse-quality-inspection-core
- Scope: Table, Enum, Codeunit (data model + business logic)
- Dependencies: None
- Estimated phases: 2

### Spec B: warehouse-quality-inspection-ui
- Scope: Pages, FactBox, Actions
- Dependencies: Spec A must be completed first
- Estimated phases: 2

Order: Spec A → Spec B (sequential)
​```

### Architecture Document Sections (MANDATORY for MEDIUM/HIGH)
1. Executive Summary
2. Business Context (Problem + Success Criteria)
3. Solution Architecture (with Mermaid diagrams: data flow, process flow, object relationships)
4. Data Model (tables, extensions, enums — high level with relationships)
5. Business Logic (codeunits, event architecture)
6. User Interface (pages, factboxes, reports — layout design)
7. Integration Points (events, APIs, external systems)
8. Security Model (permission sets, data classification)
9. Performance Considerations (hotspots, optimization strategy)
10. Technical Decisions (min 3, with alternatives and rationale)
11. Implementation Phases (ordered, with dependencies)
12. Risks & Mitigations (min 3)
13. Deployment Plan (pre/post checklist)
14. Spec Decomposition (if applicable — defines which specs to create)
```

El architect handoff debe ser a `al-spec.create`, NO al conductor:
```markdown
## Next Steps

> **Architecture Approved — Create Technical Specifications**
>
> The solution architecture is approved. Create the detailed technical spec(s):
>
> If single spec:
> ```
> @workspace use al-spec.create
> Create spec for {req_name}. Read .github/plans/{req_name}.architecture.md
> ```
>
> If decomposed (multiple specs):
> ```
> @workspace use al-spec.create
> Create spec for {req_name}-core. Read .github/plans/{req_name}.architecture.md section "Spec Decomposition"
> ```
> Then repeat for each sub-spec.
>
> After all specs are created, proceed to implementation:
> ```
> @al-conductor
> Implement {req_name}. Contracts in .github/plans/
> ```
```

## 8B. Restaurar al-spec.create.prompt.md como herramienta de detalle técnico

Lee el fichero original v2.11.0: `archive/v2.11.0/prompts/al-spec.create.prompt.md`

El spec.create genera el DETALLE TÉCNICO implementable. Debe incluir:

- Objetos AL detallados (Table, Codeunit, Page, Enum, Report) con IDs y nombres
- Campos por tabla con tipo, longitud, FlowField, CalcFormula si aplica
- Campos expuestos en pages con propiedades
- Procedimientos de codeunits con parámetros
- Event subscribers/publishers con firma exacta
- API endpoints si aplica
- Código AL de ejemplo para lógica compleja
- Acceptance criteria técnicos (compilación, tests, naming)

Cambios vs v2.11.0:
1. Output path: `.github/specs/` → `.github/plans/{req_name}.spec.md` (convención v1.1)
2. DEBE leer `.github/plans/{req_name}.architecture.md` si existe (input del architect)
3. DEBE leer `memory.md` para contexto del proyecto
4. Handoff: al final indica pasar a `@al-conductor` (no a architect)

```markdown
## Next Steps

> **Technical Specification Complete — Ready for Implementation**
>
> For MEDIUM/HIGH complexity (with architecture):
> ```
> @al-conductor
> Implement {req_name}.
> Contracts in .github/plans/:
> - {req_name}.architecture.md
> - {req_name}.spec.md
> Complexity: MEDIUM/HIGH
> ```
>
> For LOW complexity (without architecture):
> ```
> @al-developer
> Implement {req_name}. Read .github/plans/{req_name}.spec.md
> ```
```

IMPORTANTE: Cuando el spec.create recibe un requerimiento directamente del usuario (sin architecture.md), 
debe generar la spec técnica con el nivel de detalle que pueda. Pero cuando recibe input del architect 
(architecture.md existe), debe ALINEAR la spec con las decisiones del architect: IDs, patterns, 
event design, etc. NO contradecir el architecture.md.

## 8C. Actualizar docs/framework/ALDC-Core-Spec-v1.1.md

Corregir la sección de flujo:

```markdown
## Flujo de creación de artefactos

### MEDIUM/HIGH Complexity
1. `@al-architect`: Solution Architect — analiza requerimiento de negocio, diseña solución
   - Genera `.github/plans/{req_name}.architecture.md`
   - Diseña flujos de información, procesos, objetos (alto nivel)
   - Toma decisiones estratégicas con alternativas documentadas
   - Puede descomponer en múltiples specs si la complejidad lo requiere
   - GATE: usuario aprueba architecture
2. `al-spec.create`: Spec técnica — detalla el blueprint implementable
   - Lee architecture.md como input
   - Genera `.github/plans/{req_name}.spec.md` con objetos AL, IDs, campos, código
   - Blueprint que el implement-subagent puede ejecutar directamente
3. `@al-conductor`: Orquestador TDD — implementa contra los contratos
   - Lee spec.md + architecture.md
   - Ciclo: Plan → TDD Implement (via al-implement-subagent) → Review → Commit
   - GATE: usuario aprueba cada fase
4. Actualizar `memory.md` con resultado

### LOW Complexity
1. `al-spec.create`: Spec técnica directa (sin architect)
2. `@al-developer`: Implementación directa

### Resumen de roles
| Primitiva | Rol | Qué genera | Analogía |
|-----------|-----|------------|----------|
| @al-architect | Solution Architect | architecture.md (diseño, flujos, decisiones) | El que diseña el edificio |
| al-spec.create | Spec Técnica | spec.md (objetos, campos, código) | El que detalla los planos |
| @al-conductor | Orquestador TDD | Código AL + tests | El director de obra |
| al-implement-subagent | Developer TDD | Objetos AL con tests | El albañil especializado |
| al-review-subagent | Reviewer | Veredicto de calidad | El inspector |
```

## 8D. Actualizar copilot-instructions.md

```markdown
## Flujo por complejidad

LOW:
  al-spec.create → @al-developer

MEDIUM/HIGH:
  @al-architect → al-spec.create → @al-conductor

El architect puede descomponer un requerimiento complejo en múltiples specs.
Cada spec se implementa con su propio ciclo de conductor.
```

## 8E. Actualizar QUICKSTART.md

Corregir "Flujo por requerimiento" con el modelo correcto:
- MEDIUM: @al-architect → al-spec.create → @al-conductor
- Explicar que el architect puede dividir requerimientos

## 8F. Actualizar al-conductor.agent.md — Prerequisites

```markdown
### Recommended Workflow

For LOW complexity (isolated changes):
  al-spec.create → @al-developer

For MEDIUM complexity (2-3 phases, internal integrations):
  @al-architect → al-spec.create → @al-conductor

For HIGH complexity (4+ phases, architectural decisions):
  @al-architect → al-spec.create (may produce multiple specs) → @al-conductor (per spec)

For SPECIALIZED domains:
  - API: @al-architect (loads skill-api) → al-spec.create → @al-conductor
  - Copilot: @al-architect (loads skill-copilot) → al-spec.create → @al-conductor
  - Performance: @al-architect (loads skill-performance) → al-spec.create → @al-conductor
```

## 8G. Actualizar Manifesto

Añadir claridad sobre los roles:
```markdown
## Modelo de roles

ALDC refleja cómo trabaja un equipo real de desarrollo Business Central:

- **al-architect** = Solution Architect — Diseña la solución, toma decisiones estratégicas, 
  puede descomponer requerimientos complejos. Genera el DISEÑO.
- **al-spec.create** = Especificación Técnica — Detalla el blueprint implementable con 
  objetos AL, campos, código. Genera los PLANOS.
- **al-conductor** = Director de Obra — Orquesta la implementación TDD con subagents 
  especializados. EJECUTA los planos.
- **al-developer** = Developer — Implementa directamente para tareas LOW o ajustes post-entrega.
```
