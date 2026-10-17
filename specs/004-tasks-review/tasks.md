---
description: "Task list for feature implementation: Revisión de Tasks (eliminar lectura del plan)"
---

# Tasks: Revisión de Tasks (eliminar lectura del plan)

**Input**: Design documents from `/specs/004-tasks-review/`

**Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `contracts/cli.md`, `quickstart.md`

**Tests**: No se solicitan tests (la verificación es `npm run typecheck` + pruebas manuales, per constitución Principio VI).

**Nota de acoplamiento**: US1, US2 y US3 comparten los mismos archivos (`charlIA/src/index.ts`, `nodes.ts`, `opencode.ts`). US1 es el refactor central; US2 y US3 son refinamientos sobre la misma superficie (parseo de CLI y validación de `path`).

**Organización**: Tareas agrupadas por user story para trazabilidad e implementación/validación incremental.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivos distintos, sin dependencias)
- **[Story]**: User story al que pertenece (US1, US2, US3)
- Rutas exactas en cada descripción. El código vive en el submódulo `charlIA/`.

---

## Phase 1: Setup

**Purpose**: Confirmar entorno de ejecución (no hay inicialización de proyecto nueva).

- [X] T001 Verificar entorno: `npm install` en `charlIA/` y confirmar que el CLI `opencode` está en el PATH

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Tipos y helper de review compartidos por todos los user stories.

**⚠️ CRITICAL**: Ningún user story puede implementarse hasta completar esta fase.

- [X] T002 [P] Actualizar `charlIA/src/opencode.ts`: renombrar `OpenCodeReviewInput` a `{ tasksContent, cwd }`, `buildReviewPrompt(planContent)` → `buildReviewPrompt(tasksContent)`, reescribir `PROMPT_TEMPLATE` para revisar "las tasks" (claridad, completitud, riesgos, pasos faltantes, en español) y actualizar `runOpenCodeReview`
- [X] T003 [P] Actualizar tipos en `charlIA/src/nodes.ts`: `SpecReaderInput = { path, tasks }`, `SpecReaderState = { path, tasks, tasksPath, tasksContent, tasksReview? }` (eliminar `plan*` y `planReview`) y `SpecReaderPaths = Pick<SpecReaderState, "path" | "tasks">`

**Checkpoint**: Tipos y helper de review listos — comienzan los user stories.

---

## Phase 3: User Story 1 - Revisar tasks con solo un path (Priority: P1) 🎯 MVP

**Goal**: El agente acepta solo `path`, lee `tasks.md` por defecto, lo revisa y devuelve `tasksPath`, `tasksContent` y `tasksReview`. Ya no lee ningún plan.

**Independent Test**: `npm run start -- <spec-folder>` devuelve el estado con `tasksPath`, `tasksContent` y `tasksReview`, sin error por plan inexistente.

### Implementation for User Story 1

- [X] T004 [US1] Actualizar `readInput` en `charlIA/src/nodes.ts` para devolver `{ path, tasks }` (depende de T003)
- [X] T005 [US1] Actualizar `readFiles` en `charlIA/src/nodes.ts` para resolver `tasksPath = join(path, tasks)` y leer solo `tasksContent`; envolver el error `ENOENT` con mensaje legible que incluya la ruta resuelta (depende de T003)
- [X] T006 [US1] Renombrar `reviewPlan` → `reviewTasks` en `charlIA/src/nodes.ts`: si `tasksContent.trim()` está vacío devolver `tasksReview` informativo sin llamar al LLM; si no, `runOpenCodeReview({ tasksContent, cwd: path })` y guardar `tasksReview` (depende de T002, T003)
- [X] T007 [US1] Actualizar `charlIA/src/index.ts`: importar `reviewTasks`, encadenar `readInput → readFiles → reviewTasks`, parsear `const [, , path, tasks = "tasks.md"] = process.argv`, actualizar mensaje de uso a `npm run start -- <path> [tasks]`, e invocar `agent.invoke({ path, tasks })` (depende de T004, T005, T006)

**Checkpoint**: US1 funcional — `path` solo produce review de `tasks.md`.

---

## Phase 4: User Story 2 - Elegir un nombre de archivo de tasks distinto (Priority: P2)

**Goal**: Poder indicar un nombre de archivo de tasks distinto a `tasks.md`.

**Independent Test**: `npm run start -- <spec-folder> <otro-nombre.md>` lee ese archivo en lugar del default.

### Implementation for User Story 2

- [X] T008 [US2] Verificar que el argumento `tasks` fluye de `index.ts` a `readFiles` (vía `state.tasks`) y que `tasksPath` se resuelve con el nombre explícito; validar end-to-end con un archivo de nombre no estándar (depende de T007)

**Checkpoint**: US1 y US2 funcionan — nombre de tasks explícito soportado.

---

## Phase 5: User Story 3 - Mensaje de error claro ante path vacío (Priority: P3)

**Goal**: `path` omitido o vacío muestra un mensaje de uso claro y termina con código distinto de cero.

**Independent Test**: `npm run start --` (sin `path`) imprime el mensaje de uso y termina con código distinto de cero.

### Implementation for User Story 3

- [X] T009 [US3] En `charlIA/src/index.ts`, rechazar `path` vacío o solo espacios con `console.error("Uso: npm run start -- <path> [tasks]")` + `process.exit(1)`; y en `readInput` de `charlIA/src/nodes.ts` lanzar error si `path.trim()` es vacío (depende de T004, T007)

**Checkpoint**: Los tres user stories funcionan de forma independiente.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final y documentación de validación manual.

- [X] T010 Ejecutar `npm run typecheck` en `charlIA/` y confirmar que concluye sin errores (SC-004)
- [X] T011 [P] Ejecutar las validaciones manuales de `specs/004-tasks-review/quickstart.md`: (a) path solo, (b) nombre de tasks explícito, (c) path vacío, (d) archivo de tasks inexistente, (e) archivo de tasks vacío

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Sin dependencias — comienza de inmediato.
- **Foundational (Phase 2)**: Depende de Setup — BLOQUEA todos los user stories.
- **User Stories (Phase 3+)**: Dependen de Foundational. US2 y US3 dependen de US1 (misma superficie de archivos).
- **Polish (Phase 6)**: Depende de todos los user stories.

### User Story Dependencies

- **User Story 1 (P1)**: Tras Foundational — sin dependencia de otros stories.
- **User Story 2 (P2)**: Depende de US1 (comparte `index.ts`).
- **User Story 3 (P3)**: Depende de US1 (comparte `index.ts`/`nodes.ts`).

### Within Each User Story

- Tipos (`nodes.ts`) antes que nodos.
- Helper de review (`opencode.ts`) antes que `reviewTasks`.
- `readInput`/`readFiles`/`reviewTasks` antes que el encadenamiento en `index.ts`.

### Parallel Opportunities

- T002 y T003 ([P]) pueden ejecutarse en paralelo (archivos distintos).
- T011 ([P]) puede ejecutarse en paralelo con la revisión de código.

---

## Parallel Example: Foundational

```bash
# En paralelo (archivos distintos):
Task: "Actualizar charlIA/src/opencode.ts ..." (T002)
Task: "Actualizar tipos en charlIA/src/nodes.ts ..." (T003)
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Completar Phase 1: Setup.
2. Completar Phase 2: Foundational (CRÍTICO — bloquea todo).
3. Completar Phase 3: US1.
4. **PARAR y VALIDAR**: `npm run start -- <spec-folder>` devuelve review de tasks.
5. Ejecutar `npm run typecheck`.

### Incremental Delivery

1. Setup + Foundational → base lista.
2. Agregar US1 → validar → (MVP).
3. Agregar US2 → validar con nombre explícito.
4. Agregar US3 → validar path vacío.
5. Polish: typecheck + validaciones manuales del quickstart.

---

## Notes

- [P] = archivos distintos, sin dependencias.
- [Story] mapea la tarea al user story para trazabilidad.
- Cada user story debe poder validarse de forma independiente.
- Commit tras cada tarea o grupo lógico.
- Evitar: tareas vagas, conflictos sobre el mismo archivo, dependencias entre stories que rompan independencia.
