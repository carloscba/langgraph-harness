# Research: Agente implementador TDD determinista

**Feature**: `005-tdd-implementing-agent`
**Date**: 2026-10-17

## Decisión: Adaptador común para `opencode`

**Decision**: Extender el patrón de `charlIA/src/opencode.ts` a un adaptador estructurado por operación. Cada operación recibe `cwd`, prompt, modelo y perfil de permisos; `opencode run --format json` devuelve el `sessionID` y `opencode export <sessionID>` devuelve el texto o JSON de la sesión.

**Rationale**: El código actual ya captura stdout/stderr, extrae el último mensaje assistant y trabaja con `cwd`. Centralizarlo evita duplicar parsing, timeouts y manejo de errores en Test Writer, parser, implementador, marker y refactor.

**Alternatives considered**:

- Usar `@langchain/openai` directamente: descartado porque no proporciona el acceso intencional a `opencode-go`, edición contextual ni el perfil de permisos del CLI requerido por la feature.
- Mantener una función separada por nodo: descartado por duplicación y por dificultar la aplicación consistente de permisos y timeouts.

## Decisión: permisos de escritura acotados

**Decision**: Las operaciones que modifican el repositorio se ejecutan con `cwd` igual al repositorio objetivo y un perfil que permite edición únicamente dentro de ese repositorio. El adaptador no reutilizará ciegamente el perfil read-only actual.

**Rationale**: El wrapper existente deniega `edit` y `bash` y además usa `--dangerously-skip-permissions`; eso no satisface la escritura requerida ni el aislamiento solicitado. El plan conserva el aislamiento en un único punto.

**Alternatives considered**:

- Permitir edición global: descartado por riesgo de modificar archivos fuera de `path`.
- Hacer que el agente escriba archivos directamente: descartado para los nodos LLM porque la spec exige delegar esas operaciones en `opencode`.

## Decisión: resolver el runner desde `package.json`

**Decision**: Leer únicamente `<path>/package.json`. Seleccionar `scripts.test` si existe; de lo contrario, seleccionar el primer script disponible entre `vitest` y `jest`. Ejecutar con `npm run <script> -- <testPath>` usando `spawn` con argumentos y `shell: false`.

**Rationale**: Mantiene la selección determinista y respeta el script configurado por el repositorio. La terminal, no el LLM, determina el resultado.

**Alternatives considered**:

- Inferir el runner desde dependencias instaladas: descartado por ser menos determinista.
- Construir un comando shell: descartado por quoting inseguro y dificultad para preservar rutas.

## Decisión: contrato estructurado de resultados

**Decision**: Cada ejecución retorna `exitCode`, `signal`, `stdout`, `stderr`, `timedOut`, error de spawn, comando, `cwd`, target y `passed`. `passed` solo es verdadero con código 0, sin timeout y sin error de spawn.

**Rationale**: El implementador necesita el stderr exacto y el orquestador necesita distinguir fallo de test, timeout, comando ausente y proceso terminado por señal.

**Alternatives considered**:

- Rechazar la Promise ante timeout/spawn: descartado porque perdería los diagnósticos necesarios para reintentar o reportar.
- Inferir éxito del stdout: descartado; contradice la terminal como fuente de verdad.

## Decisión: tareas agnósticas al formato

**Decision**: El parser de tareas y el marcador de completadas son operaciones separadas de `opencode` con salida JSON estricta. El código TypeScript solo valida IDs, orden, descripciones, estados y confirmaciones; no interpreta checkboxes, listas ni sintaxis específica.

**Rationale**: La agnosticidad al formato es un pilar del producto. La normalización permite que el orquestador sea determinista sin acoplarse a Markdown o spec-kit.

**Alternatives considered**:

- Regex para checkboxes/listas: descartado porque falla con formatos propietarios o archivos no Markdown.
- Aceptar texto libre del LLM como estado interno: descartado porque un error de interpretación podría marcar la tarea equivocada.

## Decisión: semántica de reintentos

**Decision**: El RED inicial es obligatorio y no consume `maxRetries`. `maxRetries` cuenta intentos de implementación/fix posteriores al RED; cada intento vuelve a correr el mismo `testPath`. El valor por defecto es 5 y `0` detiene después de verificar RED.

**Rationale**: Separa la validación TDD de los intentos de reparación y hace el límite observable y testeable.

**Alternatives considered**:

- Contar el RED dentro de `maxRetries`: descartado porque hace ambiguo el número real de intentos de implementación.
- Reintentar también escribir la prueba: descartado porque el MVP define una prueba por tarea y un path estable.

## Decisión: refactor único al final

**Decision**: Ejecutar una sola operación de refactor sobre el conjunto de cambios después de que todas las tareas pendientes estén GREEN y marcadas. Luego ejecutar una corrida final de tests; si falla, reportar regresión y no continuar.

**Rationale**: La decisión explícita de la spec busca optimizar el conjunto y evitar refactors redundantes por tarea.

**Alternatives considered**:

- Refactor por tarea: descartado por redundancia y menor contexto global.
