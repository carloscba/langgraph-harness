# CLI Contract: Agente implementador TDD

## Invocation

Desde `charlIA/`:

```bash
npm run start -- <path> [tasks] [maxRetries]
```

| Argumento | Requerido | Default | Regla |
|---|---:|---|---|
| `path` | Sí | - | Raíz del repositorio objetivo; vacío es error. |
| `tasks` | No | `tasks.md` | Nombre/path relativo del archivo de tareas dentro de `path`. |
| `maxRetries` | No | `5` | Entero no negativo; cuenta fixes posteriores al RED. |

No se aceptan commits ni opciones que cambien el repositorio Git. La única persistencia intencional es el código/pruebas generado y el estado de la tarea.

## Output

El proceso imprime un objeto JSON en stdout:

```json
{
  "status": "completed",
  "tasksPath": "/repo/tasks.md",
  "testCommand": {
    "executable": "npm",
    "args": ["run", "test", "--", "tests/example.test.ts"],
    "scriptName": "test"
  },
  "completedTasks": ["task-1", "task-2"],
  "refactor": {
    "performed": true,
    "finalTest": {
      "passed": true,
      "exitCode": 0
    }
  }
}
```

Los resultados reales de test conservan `stdout`, `stderr`, `signal`, `timedOut` y `spawnError` aunque el ejemplo los omita.

Valores posibles de `status`:

- `completed`: todas las tareas pendientes pasaron, fueron marcadas y el refactor final no regresionó.
- `failed`: una tarea, el marcado o el refactor falló; incluye el último diagnóstico.
- `no-tasks`: archivo vacío o sin tareas interpretables; no hubo llamadas LLM ni escrituras.
- `no-test-command`: no se encontró `test`, `vitest` ni `jest` en `package.json`.

## Failure behavior

- Falta `path`: imprimir uso en stderr y terminar con código distinto de cero.
- Archivo inexistente, JSON inválido, tarea ambigua o path de test fuera del repositorio: detener antes de la siguiente escritura.
- RED que no falla: `failed` con error `red-not-observed`; no implementar ni marcar.
- `maxRetries` agotado: `failed` con el último `TestResult`; no procesar tareas posteriores.
- Fallo del marcador tras GREEN: `failed`; no continuar silenciosamente.
- Fallo de test final después del refactor: `failed` con regresión; no alterar estados de tareas ya confirmados.

## Internal operation contracts

### Parser

`opencode` debe devolver JSON con `sourcePath` y `tasks[]`, donde cada tarea contiene `id`, `ordinal`, `description` y `completed`. El código valida la forma, unicidad y orden; no interpreta sintaxis de tareas.

### Test Writer

Debe devolver exactamente un `testPath` y opcionalmente `affectedFile`. El path debe existir después de la operación y resolverse dentro de `path`.

### Implementor/Fixer

Recibe solo la descripción/ID de tarea, contenido o path de prueba, archivo afectado y el último `TestResult` exacto, especialmente `stderr`. No recibe el repositorio completo ni resultados no relacionados.

### Marker

Recibe `taskId`, descripción y `tasksPath`. Debe modificar solo el estado de esa tarea, preservar el formato y devolver confirmación con el mismo ID.
