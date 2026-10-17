# CLI Contract — 004-tasks-review

El agente expone una interfaz de línea de comandos (CLI).

## Invocación

```bash
npm run start -- <path> [tasks]
```

| Argumento | Posición | Requerido | Default | Descripción |
|-----------|----------|-----------|---------|-------------|
| `path` | 1 | Sí | — | Carpeta de la spec a revisar |
| `tasks` | 2 | No | `tasks.md` | Nombre del archivo de tasks |

## Comportamiento de salida

### Éxito (código 0)

Imprime el estado final como JSON (`JSON.stringify(result, null, 2)`) con la forma:

```json
{
  "path": "<carpeta>",
  "tasks": "tasks.md",
  "tasksPath": "<ruta resuelta>",
  "tasksContent": "<contenido>",
  "tasksReview": "<feedback>"
}
```

### Error (código distinto de 0)

- `path` vacío u omitido → mensaje de uso en `stderr`:
  `Uso: npm run start -- <path> [tasks]`
- Archivo de tasks inexistente → error legible con la ruta resuelta.
- Fallo de `opencode run`/`export` → error con código y `stderr` (comportamiento existente, sin cambios).

## Notas

- No se acepta ningún argumento de plan (FR-003).
- El mensaje de uso y el prompt de revisión no mencionan "plan" (SC-002).
