# Research — 004-tasks-review

Ningún punto del Technical Context quedó como "NEEDS CLARIFICATION": el feature es
un refactor acotado sobre código existente y todas las decisiones técnicas ya están
resueltas por la constitución y el estado actual del agente. Este archivo consolida
las decisiones clave y las alternativas descartadas.

## Decisiones

### 1. Estado del agente: solo tasks

- **Decision**: `SpecReaderState` queda como `{ path, tasks, tasksPath, tasksContent, tasksReview? }`. Se eliminan `plan`, `planPath`, `planContent` y `planReview`.
- **Rationale**: El spec (FR-003, FR-008) exige eliminar toda lectura del plan y exponer únicamente datos de tasks en el estado.
- **Alternatives considered**: Mantener `planContent` en el estado por compatibilidad. Rechazado: contradice SC-002 ("no menciona 'plan' en entrada, estado ni prompt").

### 2. Nodo de revisión renombrado

- **Decision**: Renombrar `reviewPlan` → `reviewTasks` en `src/nodes.ts` y `index.ts`.
- **Rationale**: Coherencia con el dominio (se revisan tasks) y con SC-002.
- **Alternatives considered**: Conservar el nombre `reviewPlan` internamente. Rechazado: dejaría residuos del término "plan" en el código, confundiendo mantenimiento.

### 3. Comportamiento ante tasks vacío

- **Decision**: Si `tasksContent.trim()` es vacío, `reviewTasks` NO llama a `opencode`; devuelve `tasksReview = "No hay contenido que revisar: el archivo de tasks está vacío."` y el proceso termina con código 0.
- **Rationale**: Decisión de `/speckit.clarify` (Session 2026-10-17). Evita una llamada al LLM sin sentido y da feedback claro.
- **Alternatives considered**: (a) Enviar el contenido vacío al LLM; (b) fallar con error. Rechazadas por decisión explícita del usuario.

### 4. Error ante archivo de tasks inexistente

- **Decision**: Envolver el error `ENOENT` de `readFile` en `readFiles` con un mensaje legible en español que indique el archivo y la ruta resuelta.
- **Rationale**: Edge case del spec ("Se debe devolver un error legible"). Mejora el diagnóstico frente al mensaje crudo de Node.
- **Alternatives considered**: Dejar que el error crudo de `readFile` propague. Rechazado: no es legible para el usuario final.

### 5. Validación de `path` vacío o solo espacios

- **Decision**: Doble validación: (a) en `index.ts` antes de `invoke` (mensaje de uso + `process.exit(1)`, FR-007/US3), y (b) en `readInput` (`throw` si `path.trim()` vacío) como red de seguridad para invocaciones programáticas.
- **Rationale**: US3 exige mensaje claro + código de salida distinto de cero. La validación del nodo cubre el caso en que `readInput` se use sin pasar por el CLI.
- **Alternatives considered**: Solo validar en el CLI. Rechazado: dejaría el nodo vulnerable si se invoca de otra forma.

### 6. Prompt de revisión

- **Decision**: Actualizar `PROMPT_TEMPLATE` en `opencode.ts` para pedir revisar "las tasks" (claridad, completitud, riesgos, pasos faltantes), manteniendo el formato y el idioma (español) previos.
- **Rationale**: La suposición del spec indica que el formato de feedback se mantiene, aplicado ahora a tasks.
- **Alternatives considered**: Cambiar el agente `@explore` o el modelo. Rechazado: fuera de alcance del feature.

## Mejores prácticas verificadas

- **ESM/NodeNext**: imports relativos con `.js` (Principio III).
- **Functional API**: `task()` para nodos y `entrypoint()` para el grafo (Principio I); no se reintroduce `StateGraph`.
- **Nombrado del estado**: el campo opcional `tasksReview?` se completa solo en el nodo final (patrón ya usado con `planReview?`).
- **Sin storage**: el agente sigue siendo stateless (Principio VI / Technology Constraints).
