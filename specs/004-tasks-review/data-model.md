# Data Model — 004-tasks-review

## Entidades del feature

### Archivo de tasks

Única entrada de contenido que el agente lee y revisa.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `path` | `string` | Carpeta de la spec a revisar (obligatorio, no vacío, no solo espacios) |
| `tasks` | `string` | Nombre del archivo de tasks (default `tasks.md`) |
| `tasksPath` | `string` | Ruta resuelta: `join(path, tasks)` |
| `tasksContent` | `string` | Contenido leído del archivo de tasks (UTF-8) |

### Review de tasks

Feedback textual generado por la revisión.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `tasksReview` | `string` | Feedback sobre claridad, completitud, riesgos y pasos faltantes |

## Estado del agente (`SpecReaderState`)

```ts
interface SpecReaderState {
  path: string;
  tasks: string;
  tasksPath: string;
  tasksContent: string;
  tasksReview?: string; // opcional hasta que reviewTasks lo complete
}
```

## Entrada (`SpecReaderInput`)

```ts
interface SpecReaderInput {
  path: string;
  tasks: string;
}
```

## Transiciones

1. `readInput(input)` → valida `path` y devuelve `{ path, tasks }`.
2. `readFiles(state)` → resuelve `tasksPath`, lee `tasksContent` (error legible si no existe).
3. `reviewTasks(state)` →
   - Si `tasksContent.trim() === ""` → `tasksReview` = mensaje informativo (sin LLM).
   - Si no → `tasksReview` = `runOpenCodeReview({ tasksContent, cwd: path })`.

## Validaciones

- `path` no vacío y no solo espacios (FR-007) → error + salida distinta de cero.
- Archivo de tasks inexistente → error legible con ruta resuelta.
- Archivo de tasks vacío → no se llama al LLM; `tasksReview` informativo.
