---
description: "Task list for feature implementation: Agente implementador TDD determinista (Spec-Driven)"
---

# Tasks: Agente implementador TDD determinista (Spec-Driven)

**Input**: Design documents from `/specs/005-tdd-implementing-agent/`

**Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `contracts/cli.md`, `quickstart.md`

**Tests**: No se solicitan tests unitarios del agente (per constitución Principio VI). La verificación es `npm run typecheck` en `charlIA/` más validación manual sobre repositorios objetivo con runner real. El runner de tests que implementa el agente **sí** es la fuente de verdad del ciclo Red-Green sobre el repo destino.

**Nota de acoplamiento**: US1→US5 comparten `charlIA/src/nodes.ts` e `index.ts`. US1 es el núcleo (Red-Green); US2 orquesta sobre US1; US3 y US4 son fases/comportamientos sobre la misma superficie; US5 es el parseo de CLI. Los contratos (`types.ts`), el runner (`test-runner.ts`) y el adaptador (`opencode.ts`) son foundational y bloquean todo.

**Organización**: Tareas agrupadas por user story para trazabilidad e implementación/validación incremental.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivos distintos, sin dependencias)
- **[Story]**: User story al que pertenece (US1…US5)
- Rutas exactas en cada descripción. El código vive en el submódulo `charlIA/`.

---

## Phase 1: Setup

**Purpose**: Confirmar entorno y preparar el fixture de validación manual (no hay inicialización de proyecto nueva; `charlIA` ya existe).

- [X] T001 Verificar entorno: `npm install` en `charlIA/`, confirmar que el CLI `opencode` está en el PATH y autenticado para modelos `opencode-go`, y ejecutar `npm run typecheck` antes de los cambios para tener una baseline limpia
- [X] T002 [P] Preparar un repositorio fixture de validación en `/tmp/tdd-agent-fixture/` con `package.json` (script `test` que ejecute un runner real, p. ej. `vitest` o `node --test`), un archivo de tareas de ejemplo **sin** checkboxes de spec-kit y al menos una tarea pendiente

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Contratos internos y las dos primitivas compartidas por todos los user stories (runner puro y adaptador `opencode`).

**⚠️ CRITICAL**: Ningún user story puede implementarse hasta completar esta fase.

- [X] T003 [P] Crear `charlIA/src/types.ts` con los contratos internos: `RunInput { path, tasks, maxRetries }`, `TaskDescriptor { id, ordinal, description, completed }`, `TaskManifest { sourcePath, tasks }`, `TestCommand { executable, args, scriptName }`, `TestResult { command, cwd, targetPath, exitCode, signal, stdout, stderr, timedOut, spawnError?, passed }`, `TaskOutcome` (completed/failed/skipped) y `RunResult { status, tasksPath, testCommand?, completedTasks, failedTask?, refactor? }`
- [X] T004 [P] Crear `charlIA/src/test-runner.ts` con el resolver determinista del comando de test desde `<path>/package.json`: priorizar `scripts.test`, luego `vitest`, luego `jest`; tratar un script vacío como ausente; devolver `TestCommand` o un resultado tipado `no-test-command`
- [X] T005 [P] Implementar la ejecución pura en `charlIA/src/test-runner.ts`: `spawn` con array de args y `shell: false` ejecutando `npm run <script> -- <testPath>`; capturar `exitCode`, `signal`, `stdout`, `stderr`, `timedOut` y `spawnError`; `passed` solo si `exitCode === 0`, sin timeout y sin spawn error; timeout configurable (default 5 min) con SIGTERM→SIGKILL y sin doble resolución ante carreras de eventos
- [X] T006 [P] Refactorizar `charlIA/src/opencode.ts` a un adaptador por operación: `runOperation({ prompt, cwd, model, permissionProfile, expectJson })` que ejecute `opencode run --format json` y `opencode export <sessionID>` **ambos con el mismo `cwd`**, devolviendo resultado estructurado (`sessionId`, `model`, `exitCode`, texto/JSON, `stderr`) en lugar de lanzar ante timeout o spawn error
- [X] T007 [P] Definir en `charlIA/src/opencode.ts` los perfiles de permisos y la resolución de modelos: perfil de escritura que permita editar **solo** dentro del `path` objetivo y deniegue `external_directory` y `bash` (sin `--dangerously-skip-permissions` en operaciones de escritura); modelos vía entorno `OPENCODE_MODEL_IMPLEMENTOR` (default `opencode-go/qwen3.8-flash`), `OPENCODE_MODEL_REFACTOR` (default `opencode-go/mimo-v2.6-flash`) y `OPENCODE_MODEL_DEFAULT` para parseo/test/marcado
- [X] T008 [P] Documentar `OPENCODE_MODEL_IMPLEMENTOR`, `OPENCODE_MODEL_REFACTOR` y `OPENCODE_MODEL_DEFAULT` en `charlIA/.env.example` con valores por defecto y sin ningún secreto real

**Checkpoint**: Contratos, runner determinista y adaptador `opencode` listos — comienzan los user stories.

---

## Phase 3: User Story 1 - Ejecutar el ciclo Red-Green sobre una tarea (Priority: P1) 🎯 MVP

**Goal**: Para una tarea pendiente, el agente escribe una prueba, la ejecuta (debe fallar = RED), implementa/arregla el código mínimo hasta que pase (GREEN) y expone el path de prueba.

**Independent Test**: `npm run start -- /tmp/tdd-agent-fixture` con una sola tarea pendiente produce una prueba que falla antes de implementar y pasa después; el `RunResult` expone `testCommand`, `completedTasks` y el `TestResult` del GREEN.

### Implementation for User Story 1

- [X] T009 [US1] Actualizar `readInput` en `charlIA/src/nodes.ts` para validar `path` no vacío y normalizar `tasks` (default `tasks.md`) y `maxRetries` (entero ≥ 0, default 5), rechazando valores inválidos con mensaje claro (depende de T003)
- [X] T010 [US1] Implementar `readTaskFile` en `charlIA/src/nodes.ts`: resolver `join(path, tasks)`, leer el archivo y envolver el error `ENOENT` incluyendo la ruta resuelta; si el contenido está vacío devolver estado `no-tasks` sin invocar el LLM (depende de T003)
- [X] T011 [US1] Implementar `parseTasks` en `charlIA/src/nodes.ts`: invocar la operación `opencode` de parseo con salida JSON y validar con zod el `TaskManifest` (IDs únicos, `ordinal` secuencial, descripciones no vacías, `sourcePath` coincidente), normalizando a `TaskDescriptor[]` sin asumir formato de checkboxes (depende de T006, T007)
- [X] T012 [US1] Implementar `writeTest` en `charlIA/src/nodes.ts`: operación `opencode` que devuelva exactamente un `testPath` (más `affectedFile?` opcional), validar que el archivo existe después de escribir y que resuelve dentro de `path`, rechazando `..`, symlinks y rutas absolutas fuera de la raíz (depende de T003, T006)
- [X] T013 [US1] Implementar `runTest` en `charlIA/src/nodes.ts`: envolver `charlIA/src/test-runner.ts` con el `testPath` dirigido y devolver `TestResult` sin reinterpretar la salida (depende de T004, T005)
- [X] T014 [US1] Implementar `implementOrFix` en `charlIA/src/nodes.ts`: operación `opencode` con `OPENCODE_MODEL_IMPLEMENTOR` que reciba **solo** el ID/descripción de la tarea, el contenido o path de la prueba, el archivo afectado y el último `TestResult` exacto (especialmente `stderr`) (depende de T003, T006, T007)
- [X] T015 [US1] Implementar el bucle Red-Green en `charlIA/src/nodes.ts`: ejecutar el RED obligatorio; si la primera corrida pasa, abortar con `red-not-observed`; mientras no pase y queden intentos, invocar `implementOrFix` y re-ejecutar el **mismo** `testPath` (depende de T013, T014)
- [X] T016 [US1] Encadenar en `charlIA/src/index.ts` el flujo de una tarea (`readInput → readTaskFile → parseTasks → writeTest → runTest(RED) → implementOrFix → runTest`) conservando `entrypoint` de la Functional API y devolviendo el `RunResult` (depende de T009–T015)

**Checkpoint**: US1 funcional — una tarea recorre Red-Green completo con el runner como fuente de verdad.

---

## Phase 4: User Story 2 - Procesar múltiples tareas en secuencia (Priority: P1)

**Goal**: Procesar todas las tareas pendientes en orden, marcando cada una solo tras GREEN, y detenerse si una falla.

**Independent Test**: Con un archivo de 2+ tareas pendientes, ambas quedan marcadas y el flujo continúa en orden; con una tarea intermedia irresoluble, las siguientes no se procesan.

### Implementation for User Story 2

- [X] T017 [US2] Implementar la iteración ordenada sobre `TaskDescriptor[]` en `charlIA/src/nodes.ts`, saltando las tareas con `completed: true` sin realizar llamadas al LLM (depende de T011, T016)
- [X] T018 [US2] Implementar `markTaskComplete` en `charlIA/src/nodes.ts`: operación `opencode` que reciba `taskId`, descripción y `tasksPath`, modifique **solo** el estado de esa tarea preservando el formato original del archivo y devuelva confirmación con el mismo ID; fallar cerrado si la tarea no es identificable de forma unívoca (depende de T006, T007)
- [X] T019 [US2] Integrar el marcado en `charlIA/src/nodes.ts`: marcar la tarea como completada únicamente tras GREEN **y** confirmación del marker; si el marcado falla, detener el flujo sin continuar silenciosamente (depende de T015, T018)
- [X] T020 [US2] Detener el flujo ante una tarea fallida en `charlIA/src/nodes.ts` y no procesar las tareas restantes, conservando `failedTask` con el último `TestResult` (depende de T015, T019)
- [X] T021 [US2] Revalidar en `charlIA/src/nodes.ts` que el manifest sigue correspondiendo al archivo original (`tasksPath`) antes de cada marcado, detectando cambios externos durante la ejecución y deteniéndose si el archivo cambió (depende de T018)

**Checkpoint**: US1 + US2 funcionales — MVP completo (parsear → orquestar → escribir test → correr → implementar/fix → marcar).

---

## Phase 5: User Story 3 - Refactor único del conjunto de cambios (Priority: P2)

**Goal**: Una única pasada de refactor al final sobre el conjunto de código generado, sin alterar el comportamiento verificado, más una corrida final de tests.

**Independent Test**: Tras todas las tareas GREEN y marcadas, el agente ejecuta una sola refactorización y la corrida final pasa; si no hubo cambios, omite la fase y lo reporta.

### Implementation for User Story 3

- [X] T022 [US3] Implementar `refactorAll` en `charlIA/src/nodes.ts`: operación `opencode` con `OPENCODE_MODEL_REFACTOR` que revise estilo, tipos y simplicidad del conjunto de cambios **sin** alterar el comportamiento que las pruebas verifican (depende de T006, T007)
- [X] T023 [US3] Ejecutar la corrida final de tests tras el refactor en `charlIA/src/nodes.ts` y reportar regresión si falla, sin alterar los estados de tareas ya confirmados (depende de T005, T022)
- [X] T024 [US3] Omitir la fase de refactor en `charlIA/src/nodes.ts` cuando no hubo cambios (todas las tareas ya estaban completas al iniciar) y reportarlo en `RunResult.refactor.performed = false` (depende de T017, T022)
- [X] T025 [US3] Integrar la fase final en `charlIA/src/index.ts` después de que todas las tareas pendientes estén GREEN y marcadas (depende de T019, T023)

**Checkpoint**: US3 funcional — refactor único al final con verificación de no regresión.

---

## Phase 6: User Story 4 - Circuit breaker ante reintentos agotados (Priority: P2)

**Goal**: Limitar los reintentos de implementación con `maxRetries` (default 5), detener el flujo y reportar el último error para intervención humana.

**Independent Test**: Con `maxRetries 0` el flujo verifica RED y se detiene; con una tarea irresoluble y `maxRetries 1` se detiene tras un fix y reporta el `TestResult` exacto.

### Implementation for User Story 4

- [X] T026 [US4] Implementar la semántica de `maxRetries` en `charlIA/src/nodes.ts`: el RED obligatorio no consume reintentos; `maxRetries` cuenta intentos de `implementOrFix`; total de corridas ≤ `1 + maxRetries`; `0` detiene tras verificar RED (depende de T015)
- [X] T027 [US4] Al agotar `maxRetries` en `charlIA/src/nodes.ts`, detener el flujo y reportar `failedTask` con `id`, `testPath`, cantidad de intentos y el último `TestResult` completo (`stdout`/`stderr`/`exitCode`/`signal`/`timedOut`) (depende de T026)
- [X] T028 [US4] Rechazar `maxRetries` negativo, no entero o `NaN` en `charlIA/src/nodes.ts` con mensaje de error claro antes de cualquier llamada al LLM (depende de T009)

**Checkpoint**: Circuit breaker operativo; el flujo nunca reintenta indefinidamente.

---

## Phase 7: User Story 5 - Interfaz de línea de comandos (Priority: P3)

**Goal**: CLI con `path` obligatorio, `tasks` (default `tasks.md`) y `maxRetries` opcional, con mensaje de uso y salida JSON.

**Independent Test**: `npm run start --` sin `path` muestra uso y termina con código ≠ 0; con `path` y `maxRetries` se resuelven los defaults esperados.

### Implementation for User Story 5

- [X] T029 [US5] Parsear los argumentos CLI en `charlIA/src/index.ts`: `path` obligatorio, `tasks` (default `tasks.md`) y `maxRetries` opcional en tercera posición (depende de T016)
- [X] T030 [US5] Mostrar `Uso: npm run start -- <path> [tasks] [maxRetries]` en stderr y terminar con código ≠ 0 cuando `path` falte o sea solo espacios (depende de T029)
- [X] T031 [US5] Imprimir el `RunResult` como JSON en stdout desde `charlIA/src/index.ts` y los errores de ejecución en stderr sin filtrar secretos del entorno (depende de T029)

**Checkpoint**: Los cinco user stories funcionan de forma independiente.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final y validación manual del flujo completo.

- [X] T032 Ejecutar `npm run typecheck` en `charlIA/` y confirmar que concluye sin errores (SC-005)
- [X] T033 [P] Ejecutar las validaciones manuales de `specs/005-tdd-implementing-agent/quickstart.md`: (a) tarea única Red-Green, (b) 2+ tareas en orden, (c) formato de tareas sin checkboxes de spec-kit, (d) `stderr` exacto llegando al implementador, (e) refactor único final sin regresión
- [X] T034 [P] Validar los escenarios de fallo documentados en `specs/005-tdd-implementing-agent/quickstart.md`: sin `path`, `no-test-command`, `red-not-observed`, `maxRetries 0` y regresión en el refactor final
- [X] T035 [P] Confirmar que la ejecución no crea commits de git (`git status` en `/tmp/tdd-agent-fixture/`) y que el único cambio persistido además del código/pruebas es el estado de la tarea en el archivo de tareas (SC-001, FR-014)
- [X] T036 [P] Verificar que las escrituras de `opencode` quedaron restringidas al `path` objetivo y que no se registraron secretos en logs (Principio V)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Sin dependencias — comienza de inmediato.
- **Foundational (Phase 2)**: Depende de Setup — **BLOQUEA todos los user stories**.
- **User Stories (Phase 3+)**: Dependen de Foundational.
  - US1 y US2 son P1 y deben completarse primero (forman el MVP).
  - US3 (refactor final) e US4 (circuit breaker) dependen de US1/US2 porque operan sobre la misma superficie de nodos.
  - US5 (CLI) puede completarse en paralelo una vez que US1 define el flujo.
- **Polish (Phase 8)**: Depende de todos los user stories deseados.

### User Story Dependencies

- **US1 (P1)**: Tras Foundational — sin dependencia de otros stories.
- **US2 (P1)**: Depende de US1 (reutiliza `parseTasks`, `writeTest`, `runTest`, `implementOrFix` y el bucle).
- **US3 (P2)**: Depende de US2 (solo corre cuando no quedan tareas pendientes) y de US1 (runner).
- **US4 (P2)**: Depende de US1 (semántica del bucle de reintentos).
- **US5 (P3)**: Depende de US1 para el encadenamiento del flujo; puede afinarse en paralelo con US3/US4.

### Within Each User Story

- Contratos (`types.ts`) antes que nodos.
- Runner (`test-runner.ts`) y adaptador (`opencode.ts`) antes que los nodos que los invocan.
- `parseTasks` antes que `writeTest`.
- `runTest` antes que el bucle Red-Green.
- RED verificado antes del primer `implementOrFix`.
- Marker solo después de GREEN confirmado.

### Parallel Opportunities

- T003–T008 ([P]) pueden ejecutarse en paralelo (archivos distintos); los imports hacia `types.ts` se resuelven al integrar.
- T002 ([P]) puede ejecutarse en paralelo con el resto del Setup.
- T033, T034, T035 y T036 ([P]) pueden ejecutarse en paralelo sobre la misma ejecución validada.
- T029–T031 ([US5]) pueden afinarse en paralelo con T022–T028 una vez que T016 está listo.

---

## Parallel Example: Foundational

```bash
# En paralelo (archivos distintos):
Task: "Crear charlIA/src/types.ts con los contratos internos" (T003)
Task: "Crear charlIA/src/test-runner.ts con el resolver del comando de test" (T004)
Task: "Implementar la ejecución pura en charlIA/src/test-runner.ts" (T005)
Task: "Refactorizar charlIA/src/opencode.ts a adaptador por operación" (T006)
Task: "Definir perfiles de permisos y modelos en charlIA/src/opencode.ts" (T007)
Task: "Documentar variables de modelo en charlIA/.env.example" (T008)
```

---

## Implementation Strategy

### MVP First (US1 + US2)

1. Completar Phase 1: Setup.
2. Completar Phase 2: Foundational (CRÍTICO — bloquea todo).
3. Completar Phase 3: US1.
4. Completar Phase 4: US2.
5. **PARAR y VALIDAR**: `npm run start -- /tmp/tdd-agent-fixture` con 2+ tareas recorre Red-Green por tarea y marca cada una.
6. Ejecutar `npm run typecheck`.

### Incremental Delivery

1. Setup + Foundational → base lista.
2. Agregar US1 → validar una tarea → (MVP parcial).
3. Agregar US2 → validar múltiples tareas y marcado → (MVP completo).
4. Agregar US3 → validar refactor único final y no regresión.
5. Agregar US4 → validar circuit breaker con `maxRetries 0` y `1`.
6. Agregar US5 → validar defaults y mensaje de uso.
7. Polish: typecheck + validaciones manuales del quickstart + verificación de ausencia de commits.

---

## Notes

- [P] = archivos distintos, sin dependencias.
- [Story] mapea la tarea al user story para trazabilidad.
- Cada user story debe poder validarse de forma independiente.
- El runner de tests es la única fuente de verdad sobre RED/GREEN: nunca inferir el resultado del texto del LLM.
- No se generan commits de git en ningún punto del flujo.
- Evitar: tareas vagas, conflictos sobre el mismo archivo, dependencias entre stories que rompan independencia.
