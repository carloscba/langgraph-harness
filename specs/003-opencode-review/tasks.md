# 003-opencode-review — Tasks

- [x] Crear `src/opencode.ts`:
  - [x] Definir `buildReviewPrompt(planContent)`.
  - [x] Definir `runOpenCodeReview({ planContent, cwd })` usando `spawn`.
  - [x] Configurar args `["run", "--agent", "explore", "--model", "opencode-go/deepseek-v4-pro", prompt]`.
  - [x] Manejar `ENOENT`, códigos de salida distintos de 0 y timeout.
- [x] Actualizar `src/nodes.ts`:
  - [x] Agregar `planReview: string` a `SpecReaderState`.
  - [x] Importar `runOpenCodeReview` desde `./opencode.js`.
  - [x] Crear task `reviewPlan` que llame al helper con `planContent` y `cwd = state.path`.
- [x] Actualizar `src/index.ts`:
  - [x] Importar `reviewPlan`.
  - [x] Encadenar `readInput → readFiles → reviewPlan`.
- [x] Crear spec files en `specs/003-opencode-review/`:
  - [x] `specification.md`
  - [x] `plan.md`
  - [x] `tasks.md`
- [x] Verificar con `npm run typecheck`.
- [x] Prueba manual: `npm run start -- ./specs/002-spec-invocation`.
