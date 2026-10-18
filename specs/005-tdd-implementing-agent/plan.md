# Implementation Plan: Agente implementador TDD determinista (Spec-Driven)

**Branch**: `005-tdd-implementing-agent` | **Date**: 2026-10-17 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/005-tdd-implementing-agent/spec.md`

## Summary

Reemplazar el review pasivo de `charlIA` por un agente que procese un archivo de tareas agnóstico al formato. El agente normalizará las tareas mediante `opencode`, ejecutará Red-Green por cada tarea con un runner de tests determinista y un límite de reintentos, marcará la tarea únicamente después de un resultado GREEN y ejecutará una única fase final de refactor sobre el conjunto de cambios. El marcado de tareas también será delegado a `opencode` para preservar el formato original.

La implementación reutilizará el adaptador de subprocesos existente, pero lo convertirá en una interfaz estructurada por operación y con permisos de escritura limitados al repositorio objetivo. El runner de tests permanecerá como código puro y será la fuente de verdad sobre RED/GREEN.

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Language/Version**: TypeScript 7 sobre Node.js, ESM con `moduleResolution: NodeNext`

**Primary Dependencies**: `@langchain/langgraph` Functional API, `@langchain/core`, `zod` v4, `tsx`, `dotenv`; CLI externo `opencode`; APIs estándar de Node (`child_process`, `fs/promises`, `path`)

**Storage**: Archivos del repositorio objetivo (`tasks` y código/pruebas); sin base de datos

**Testing**: `npm run typecheck` para el agente; el runner ejecuta el script de test del repositorio objetivo mediante `npm run <script> -- <testPath>`

**Target Platform**: CLI Node.js en Linux; el repositorio objetivo debe tener `package.json` y un script `test`, `vitest` o `jest`

**Project Type**: Agente CLI LangGraph.js dentro del submódulo `charlIA`

**Performance Goals**: No se requiere throughput; cada ejecución debe respetar timeouts configurables y no iniciar trabajo LLM si no hay tareas o no existe comando de test

**Constraints**: Un test por tarea; `maxRetries` default 5; RED obligatorio antes de implementar; el runner captura stdout/stderr sin reinterpretarlos; no commits; escrituras de `opencode` limitadas al path objetivo; imports relativos con `.js`

**Scale/Scope**: Una ejecución sobre un repositorio y un archivo de tareas; cantidad arbitraria de tareas secuenciales; una prueba dirigida por tarea y una refactorización final

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Cumplimiento |
|-----------|--------------|
| I. LangGraph.js Functional API | ✅ El flujo seguirá `entrypoint` + `task` en `charlIA/src/index.ts` y `nodes.ts`; no se introduce `StateGraph`. |
| II. Spec-Driven Development | ✅ La feature tiene spec y este plan; `tasks.md` se generará después mediante `/speckit.tasks`. |
| III. ESM + NodeNext | ✅ Se mantienen ESM, NodeNext e imports relativos con extensión `.js`. |
| IV. OpenRouter-Only LLM Access | ⚠️ Excepción justificada: la feature requiere delegar operaciones de desarrollo al CLI `opencode` para usar modelos `opencode-go`; no se añadirá un cliente LLM nativo. La excepción queda aislada en el adaptador de `opencode` y no modifica `src/model.ts`. |
| V. Secrets & Environment Hygiene | ✅ No se añaden secretos; el CLI hereda el entorno existente y no se imprime ninguna key. |
| VI. Typecheck Gate & Simplicity | ✅ Se ejecutará `npm run typecheck`; se reutilizan los módulos existentes y solo se separa el runner puro por su responsabilidad determinista. |

**Resultado**: PASS con excepción documentada del Principio IV, requerida explícitamente por la spec. La alternativa OpenRouter directa no satisface el requisito de disponer de los modelos y capacidades de desarrollo de `opencode-go`.

## Project Structure

### Documentation (this feature)

```text
specs/005-tdd-implementing-agent/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (submodule `charlIA`)

```text
charlIA/
├── src/
│   ├── index.ts          # CLI, entrypoint y orquestación secuencial
│   ├── nodes.ts          # tasks LangGraph: parseo, TDD, marcado y refactor
│   ├── opencode.ts       # adaptador subprocess, prompts y permisos por operación
│   ├── test-runner.ts    # resolver y ejecución pura del comando de tests
│   └── types.ts          # contratos internos compartidos
└── package.json
```

**Structure Decision**: Single CLI agent en el submódulo `charlIA`. Se conservan `index.ts`, `nodes.ts` y `opencode.ts`; `test-runner.ts` aísla la única parte que no usa LLM y `types.ts` evita duplicar los contratos de estado entre nodos. No se crea una suite de tests dentro de `charlIA` en este MVP; la validación funcional se realiza sobre repositorios objetivo y el gate local es `typecheck`.

## Phase 0: Research Summary

Los detalles de investigación están en [research.md](./research.md). Decisiones principales:

- Usar `spawn` con arrays de argumentos y `shell: false` para el runner; el resultado siempre se devuelve estructurado, incluidos timeout y errores de spawn.
- Resolver scripts únicamente desde `package.json`, priorizando `test`, luego `vitest` y `jest`; no inferir runners escaneando dependencias.
- Normalizar tareas y marcar completadas mediante respuestas JSON estrictas de `opencode`, sin parsear checkboxes en código.
- Validar que el `testPath` devuelto por Test Writer permanezca dentro del repositorio objetivo.
- Contar `maxRetries` como intentos de implementación posteriores al RED obligatorio; `0` verifica RED y detiene.
- Aplicar el mismo `cwd` a `opencode run` y `opencode export`, y usar perfiles de permisos con escritura restringida al repositorio.

## Phase 1: Design

Los artefactos de diseño son:

- [data-model.md](./data-model.md): estado de ejecución, manifest de tareas, resultados de tests y transiciones.
- [contracts/cli.md](./contracts/cli.md): contrato de invocación CLI, salida JSON y errores.
- [quickstart.md](./quickstart.md): validación manual del flujo MVP y escenarios de fallo.

### Execution Flow

1. `readInput` valida `path`, normaliza `tasks` y `maxRetries`.
2. `readTaskFile` carga el archivo; vacío o sin tareas devuelve `no-tasks` sin llamadas LLM.
3. `parseTasks` llama a `opencode` con salida JSON estructurada y obtiene tareas ordenadas y estados.
4. Por cada tarea pendiente: `writeTest` produce exactamente un `testPath`; el runner ejecuta RED.
5. Si RED no se observa, el flujo se detiene con `red-not-observed`.
6. Mientras no pase y queden intentos: `implementOrFix` recibe solo test, archivo afectado y el resultado exacto más reciente; el runner vuelve a ejecutar el mismo `testPath`.
7. Al pasar, `markTaskComplete` actualiza solo el estado de esa tarea mediante `opencode`; si falla, se detiene sin continuar silenciosamente.
8. Cuando todas las tareas están completas, `refactorAll` ejecuta una sola revisión final; después el runner ejecuta la suite dirigida o completa según el contrato del nodo.
9. El resultado final resume tareas completadas, errores y la corrida final.

### Error Handling

- Los errores de lectura, parseo, comando de test ausente, path inválido, timeout, spawn y marcado se representan en el resultado y detienen el flujo antes de realizar escrituras posteriores.
- El runner nunca determina GREEN a partir de texto del LLM: solo `exitCode === 0`, sin timeout ni error de spawn.
- No se marcan tareas como completadas tras un fallo de test, refactor o persistencia.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| Principio IV: OpenRouter-Only LLM Access | La feature requiere el CLI `opencode` y modelos `opencode-go` para operaciones de desarrollo y edición contextual. | Usar `ChatOpenAI` vía OpenRouter no proporciona el flujo de edición/permisos del CLI ni satisface el requisito de modelos `opencode-go`; reutilizar el wrapper existente mantiene el alcance acotado. |
