# Quickstart — 004-tasks-review

## Requisitos

- Node.js + `npm`.
- CLI `opencode` instalado y en el PATH.
- `.env` con `OPENROUTER_API_KEY` y `MODEL` (ver `.env.example`).

## Uso (dentro de `charlIA/`)

```bash
# Revisar las tasks de una spec usando el nombre por defecto (tasks.md)
npm run start -- ../specs/004-tasks-review

# Revisar con un nombre de archivo de tasks explícito
npm run start -- ../specs/004-tasks-review mis-tareas.md
```

## Verificación

```bash
npm run typecheck
```

## Comportamiento esperado

- Devuelve JSON con `path`, `tasks`, `tasksPath`, `tasksContent` y `tasksReview`.
- Sin `path` → mensaje de uso y salida distinta de cero.
- Archivo de tasks inexistente → error legible.
- Archivo de tasks vacío → `tasksReview` informativo, sin llamar al LLM.
