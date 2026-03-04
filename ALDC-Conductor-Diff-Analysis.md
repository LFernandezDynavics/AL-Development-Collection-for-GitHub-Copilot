# Análisis Comparativo: Conductor v2.11.0 vs v1.1

## Resumen Ejecutivo

Los dos conductores son **casi idénticos en estructura** (862 vs 871 líneas). Las instrucciones del Core Workflow, HITL gates, style guides, stopping rules, y documentation requirements son esencialmente iguales. El problema NO está en lo que dice el conductor, sino en **lo que el modelo no ejecuta** a pesar de tenerlo escrito.

## Diferencias Textuales Reales

### 1. Frontmatter — Handoffs reducidos (CORRECTO)

**v2.11.0**: 6 handoffs → architect, API specialist, Copilot specialist, implementation specialist, testing specialist, debugging specialist
**v1.1**: 2 handoffs → architect, implementation specialist

**Veredicto**: ✅ Correcto. Los 4 handoffs eliminados apuntaban a agentes que ya no existen. Pero falta añadir handoff a `@al-developer` para "Quick Adjustments" con el nombre correcto.

### 2. Frontmatter — Tools simplificados (NEUTRAL)

**v2.11.0**: Lista exhaustiva de tools individuales (playwright, context7, microsoft-docs, etc.)
**v1.1**: Lista agrupada (`'execute'`, `'read/problems'`, `'edit'`, `'search'`, etc.)

**Veredicto**: ⚠️ Neutral pero riesgo. La lista agrupada puede hacer que Copilot no resuelva algunos tools específicos. Idealmente mantener la lista completa del v2.11.0.

### 3. Frontmatter — `agents:` explícito (NUEVO en v1.1)

**v2.11.0**: No tiene campo `agents:` en frontmatter
**v1.1**: `agents: ['al-planning-subagent', 'al-review-subagent', 'al-developer']`

**Veredicto**: ✅ Mejora. Declaración explícita de qué subagents puede invocar.

### 4. Subagent Instructions — Nombre cambiado (CORRECTO)

**v2.11.0**: `### AL Implementation Subagent` con instrucciones TDD
**v1.1**: `### @al-developer` con instrucciones TDD IDÉNTICAS

**Veredicto**: ✅ Contenido idéntico. La instrucción "Follow strict TDD: tests first (failing), minimal code, tests pass, lint/format" está presente en ambos.

### 5. Integration with Specialized Agents — Skills routing (NUEVO en v1.1)

**v2.11.0**: Referencias a agentes especializados por nombre completo
**v1.1**: Referencias a agentes simplificados + skill loading

**Veredicto**: ✅ Correcto. El routing se simplificó coherentemente con el modelo de 4 agentes + skills.

### 6. Context Requirements — Memory y naming actualizados (CORRECTO)

**v2.11.0**: `session-memory.md`, `*-arch.md`, `*-spec.md`
**v1.1**: `memory.md` (global), `*.architecture.md`, `*.spec.md`

**Veredicto**: ✅ Correcto. Alineado con convenciones v1.1.

### 7. Domain Skills — Solo en v1.1

**v1.1** añade:
```markdown
## Domain Skills
When orchestrating TDD cycles and test strategy is needed, load and follow:
- @file skills/skill-testing.md
```

**Veredicto**: ✅ Correcto pero insuficiente. Solo referencia skill-testing. Debería incluir contexto de cuándo cargar otros skills al delegar a @al-developer.

## Diagnóstico: ¿Por qué el v1.1 no ejecuta TDD/Review/Gates?

### Hallazgo crítico: El texto es (casi) idéntico pero el comportamiento es opuesto.

Ambos conductores dicen:
- "Follow strict TDD: tests first (failing), minimal code, tests pass"
- "MANDATORY STOP: Wait for user"
- "Write Phase Completion File"
- "Use #runSubagent to invoke the AL Code Review Subagent"

Pero el v1.1 NO lo hace. Las causas probables:

### Causa 1: `al-developer` como subagent no tiene TDD enforcement interno

**v2.11.0** usaba `al-implement-subagent` que era un agente **dedicado** con TDD hardcoded en su propia definición. El subagent solo sabía hacer TDD — no tenía otro modo.

**v1.1** usa `@al-developer` que es un agente **generalista** con muchos skills y modos. Cuando el conductor lo invoca como subagent, el developer puede elegir no hacer TDD porque tiene libertad operativa.

**Fix**: Reforzar en el conductor que al invocar @al-developer como subagent, MUST incluir instrucción explícita de TDD obligatorio.

### Causa 2: El modelo "optimiza" saltándose pasos que percibe como redundantes

Con un conductor más largo y un developer que ya sabe hacer las cosas, el modelo puede decidir "ya lo hago todo de una" sin parar en cada gate. Esto es un problema de LLM behavior, no de instrucciones.

**Fix**: Hacer los gates más assertivos. En vez de "MANDATORY STOP", usar lenguaje más fuerte con consecuencias explícitas.

### Causa 3: El review subagent no se invoca porque no hay errores de compilación

El modelo puede interpretar "build limpio = no necesita review" y saltarse el review subagent. Pero el review no es solo compilación — es validación contra spec, architecture, naming, patterns.

**Fix**: Hacer explícito que review MUST ejecutarse incluso con build limpio.

### Causa 4: Los phase-complete docs no se generan porque el conductor no para entre fases

Si el conductor no para (Causa 2), tampoco genera los docs intermedios.

**Fix**: Vincular el doc generation al gate: "MUST NOT proceed to next phase until phase-complete.md is written AND user confirms."

## Plan de Correcciones

### P0 — Fixes críticos en al-conductor.agent.md

#### Fix 1: Reforzar TDD enforcement al delegar a @al-developer

En la sección `### @al-developer`, después de "Instruct to:", AÑADIR:

```markdown
**CRITICAL TDD ENFORCEMENT when invoking @al-developer as subagent:**
- You MUST instruct @al-developer to create tests BEFORE implementation code
- You MUST verify that test files exist BEFORE approving implementation
- If @al-developer returns code without tests, REJECT and re-invoke with explicit TDD instruction
- The TDD cycle is: write failing test → write minimal code → verify test passes → refactor
- This is NOT optional for MEDIUM/HIGH complexity — it is a Core v1.1 MUST
```

#### Fix 2: Reforzar review subagent como OBLIGATORIO

En la sección `#### 2B. Review Implementation`, AÑADIR antes del bloque visual:

```markdown
**MANDATORY**: The review subagent MUST be invoked after EVERY phase, regardless of build status.
Build success (0 errors) does NOT exempt from review. The review validates:
- Spec compliance (object IDs, naming, field definitions match spec)
- Architecture compliance (patterns, event design match architecture)
- AL best practices (naming conventions, performance patterns)
- Test coverage (tests exist and cover the phase's acceptance criteria)
- Extension-only compliance (no base object modifications)

NEVER skip review. NEVER proceed to next phase without review verdict.
```

#### Fix 3: Reforzar MANDATORY STOP con consecuencias

En la sección `#### 2C. Return to User for Commit`, cambiar "MANDATORY STOP" a:

```markdown
4. **MANDATORY STOP — HARD GATE**: 
   - You MUST NOT invoke @al-developer for the next phase until user confirms
   - You MUST have written `.github/plans/<task-name>-phase-<N>-complete.md` BEFORE presenting checkpoint
   - You MUST show "💾 Ready to commit?" and WAIT
   - Proceeding without user confirmation is a Core v1.1 violation
   - If user says "continue" or "proceed", that counts as confirmation
```

#### Fix 4: Reforzar plan approval gate

En `### Phase 1: Planning`, después del plan presentation, AÑADIR:

```markdown
**HARD GATE — PLAN APPROVAL**:
After presenting the plan:
1. STOP and WAIT for user approval
2. DO NOT start Phase 2 until user explicitly approves
3. Present open questions and wait for answers
4. If user requests changes, revise plan and re-present
5. Only proceed when user says "approved", "proceed", "continue", "ok", or similar confirmation
```

#### Fix 5: Reforzar test-plan.md creation

En `### Phase 1: Planning`, AÑADIR:

```markdown
**Contract completeness check**:
Before starting implementation, verify the requirement set is complete:
1. `{req_name}.spec.md` — MUST exist
2. `{req_name}.architecture.md` — MUST exist for MEDIUM/HIGH
3. `{req_name}.test-plan.md` — If missing, CREATE it from `docs/templates/test-plan-template.md` during planning phase
4. `memory.md` — MUST have entry for this requirement

If test-plan.md does not exist, you MUST create it as part of Phase 1 before proceeding to implementation.
```

#### Fix 6: Reforzar memory.md update

En `### Phase 3: Plan Completion`, AÑADIR:

```markdown
**MANDATORY memory.md update**:
At plan completion, you MUST append to `.github/plans/memory.md`:
- Requirement status update (in-progress → done)
- Decisions taken during implementation
- Deviations from spec/architecture (if any)
- Lessons learned
- Next steps
This is append-only — NEVER delete existing content in memory.md.
```

### P1 — Mejoras secundarias

#### Fix 7: Ampliar Domain Skills del conductor

```markdown
## Domain Skills

When orchestrating TDD cycles and test strategy is needed, load and follow:
- @file skills/skill-testing.md

When delegating API-related implementation to @al-developer, instruct to load:
- @file skills/skill-api.md

When delegating Copilot feature implementation to @al-developer, instruct to load:
- @file skills/skill-copilot.md
```

#### Fix 8: Restaurar tools list completa del v2.11.0

Copiar la lista de tools del frontmatter v2.11.0, actualizando las referencias a agentes eliminados.

#### Fix 9: Añadir handoff a @al-developer

```yaml
handoffs:
  - label: Request Architecture Design
    agent: AL Architecture & Design Specialist
    prompt: Design architecture before implementation
  - label: Quick Adjustments
    agent: @al-developer
    prompt: Make simple adjustments after Orchestra completion
```

### P2 — Mejoras en otros componentes

#### Fix 10: Estructura directorio por requisito
Convención: `.github/plans/{req_name}/` en vez de plano.

#### Fix 11: Architecture template — más Risks y Deployment
Mínimo 3 riesgos con mitigaciones + pre/post deployment checklist.

#### Fix 12: Event design en architect
Documentar eventos candidatos (aunque sean v2).

#### Fix 13: Testing separado por tipo en architecture
Unit, Integration, API como codeunits separados.

## Ficheros a Modificar

| Fichero | Fixes | Prioridad |
|---------|-------|-----------|
| `agents/al-conductor.agent.md` | #1, #2, #3, #4, #5, #6, #7, #8, #9 | P0 |
| `docs/templates/architecture-template.md` | #11, #12, #13 | P2 |
| Spec v1.1 (contracts section) | #10 | P2 |
| `aldc.yaml` | #10 (namingConvention update) | P2 |
| Validator | #10 (subdirectory check) | P2 |
