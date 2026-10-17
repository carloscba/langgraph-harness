# Implementation Plan: Revisión de Tasks (eliminar lectura del plan)

**Branch**: `004-tasks-review` | **Date**: 2026-10-17 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/004-tasks-review/spec.md`

## Summary

Modificar el agente `charlIA` para que revise **únicamente** el archivo de tasks de una spec, eliminando por completo la lectura del plan. El agente pasará de aceptar `path` + `plan` + `tasks` a aceptar solo `path` (obligatorio) y un nombre de archivo de tasks opcional (default `tasks.md`). El estado final expone `path`, el nombre de tasks, `tasksPath`, `tasksContent` y `tasksReview`. Si el archivo de tasks está vacío, el agente no llama al LLM y devuelve un mensaje informativo.

**Enfoque técnico**: refactor del grafo LangGraph (Functional API) ya existente — renombrar `reviewPlan` → `reviewTasks`, eliminar los campos `plan*` del estado, actualizar el prompt de `opencode.ts` para revisar tasks en lugar de plan, y ajustar el parsing de argumentos en `index.ts`.

## Technical Context

**Language/Version**: TypeScript 7.0.2 sobre Node.js (ESM, `"type": "module"`)

**Primary Dependencies**: `@langchain/core`, `@langchain/langgraph` (Functional API), `@langchain/openai`, `zod` v4, `dotenv`, `tsx`

**Storage**: N/A (agente stateless; lee un archivo en el filesystem)

**Testing**: Sin framework de tests. Verificación vía `npm run typecheck` (`tsc --noEmit`) + prueba manual con `npm run start`

**Target Platform**: CLI de Node.js en Linux (submódulo `charlIA`)

**Project Type**: CLI agent (LangGraph.js)

**Performance Goals**: Sin métricas estrictas; el review debe completar dentro del timeout existente de `opencode run` (5 min)

**Constraints**: ESM + `moduleResolution: NodeNext` (imports relativos con `.js`); el CLI `opencode` debe estar instalado y en el PATH; permisos de solo-lectura en `opencode`

**Scale/Scope**: Un solo agente, lee un único archivo de tasks por ejecución

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Cumplimiento |
|-----------|--------------|
| I. LangGraph.js Functional API | ✅ Se mantiene `entrypoint` + `task` en `src/index.ts`/`src/nodes.ts` |
| II. Spec-Driven Development | ✅ Feature definida vía `/speckit.specify` + `/speckit.clarify`; este plan es la fase `plan` |
| III. ESM + NodeNext | ✅ Imports relativos con `.js` (sin cambios en este sentido) |
| IV. OpenRouter-Only LLM Access | ✅ No se modifica `src/model.ts`; el review sigue usando el CLI `opencode` existente |
| V. Secrets & Environment Hygiene | ✅ No se tocan credenciales ni `.env` |
| VI. Typecheck Gate & Simplicity | ✅ Se ejecutará `npm run typecheck`; se aplica YAGNI (solo se elimina `plan`, sin nuevas abstracciones) |

**Resultado**: PASS. Sin violaciones que justificar.

## Project Structure

### Documentation (this feature)

```text
specs/004-tasks-review/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output (CLI interface contract)
├── spec.md              # Feature specification
├── checklists/          # Spec quality checklist
└── tasks.md             # Phase 2 output (/speckit.tasks - NOT created here)
```

### Source Code (submódulo `charlIA`)

```text
charlIA/
├── src/
│   ├── index.ts         # entrypoint + parsing de argumentos CLI
│   ├── nodes.ts         # readInput, readFiles, reviewTasks
│   ├── opencode.ts      # buildReviewPrompt, runOpenCodeReview
│   └── model.ts         # (sin cambios)
├── package.json
└── tsconfig.json
```

**Structure Decision**: Single project. El agente vive en el submódulo `charlIA` (repo `carloscba/charlIA`). Los cambios de código se hacen en `charlIA/src/`; la documentación del feature se aloja en `specs/004-tasks-review/` (workspace raíz).

## Complexity Tracking

> No violations. Sin entradas.
