# 001-spec-reader — Tasks

- [x] Crear `SpecReaderState` (interface) con `path`, `plan`, `tasks`, `planPath`, `tasksPath`, `planContent`, `tasksContent`.
- [x] Crear task `readInput`:
  - [x] Recibe `{ path, plan, tasks }`.
  - [x] Valida que `path` sea string no vacío.
  - [x] Devuelve el estado inicial.
- [x] Crear task `readFiles`:
  - [x] Recibe el estado.
  - [x] Arma `planPath` y `tasksPath` con `node:path` `join`.
  - [x] Lee `plan` y `tasks` en UTF-8 con `node:fs/promises`.
  - [x] Devuelve el estado con rutas y contenidos.
- [x] Actualizar `src/index.ts`:
  - [x] Reemplazar el ejemplo `add` por el entrypoint que orquesta `readInput` → `readFiles`.
- [x] Verificar con `npm run typecheck`.