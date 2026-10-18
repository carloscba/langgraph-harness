# Data Model: Agente implementador TDD determinista

## RunInput

Entrada normalizada del agente.

| Campo | Tipo | Requerido | Validación |
|---|---|---:|---|
| `path` | `string` | Sí | No vacío; se resuelve como raíz del repositorio objetivo. |
| `tasks` | `string` | No | Default `tasks.md`; se resuelve dentro de `path`. |
| `maxRetries` | `number` | No | Entero `>= 0`; default `5`. |

## TaskDescriptor

Representación interna normalizada, independiente del formato del archivo.

| Campo | Tipo | Validación |
|---|---|---|
| `id` | `string` | Único y estable dentro del manifest. |
| `ordinal` | `number` | Secuencial y positivo. |
| `description` | `string` | No vacío; texto suficiente para generar la prueba. |
| `completed` | `boolean` | Estado interpretado por el parser. |

Relación: un `TaskManifest` contiene una lista ordenada de `TaskDescriptor` y referencia `sourcePath`.

## TaskManifest

| Campo | Tipo | Validación |
|---|---|---|
| `sourcePath` | `string` | Debe coincidir con el archivo solicitado. |
| `tasks` | `TaskDescriptor[]` | Puede estar vacío; IDs sin duplicados. |

El manifest no se persiste como archivo nuevo. Se conserva en memoria y se valida antes de cada marcado para evitar operar sobre un task ambiguo.

## TestCommand

| Campo | Tipo | Descripción |
|---|---|---|
| `executable` | `string` | MVP: `npm`. |
| `args` | `string[]` | Incluye `run`, nombre de script, `--` y opcionalmente target. |
| `scriptName` | `string` | `test`, `vitest` o `jest`. |

Resolución: `test` tiene prioridad; luego `vitest`; luego `jest`. Si ninguno existe, el estado es `no-test-command`.

## TestResult

Resultado soberano de una ejecución del runner.

| Campo | Tipo | Descripción |
|---|---|---|
| `command` | `string[]` | Comando y argumentos ejecutados. |
| `cwd` | `string` | Raíz del repositorio objetivo. |
| `targetPath` | `string` | Path de prueba dirigido, o vacío en corrida final completa. |
| `exitCode` | `number \| null` | Código del proceso. |
| `signal` | `NodeJS.Signals \| null` | Señal de terminación. |
| `stdout` | `string` | Salida exacta. |
| `stderr` | `string` | Error exacto entregado al fixer. |
| `timedOut` | `boolean` | Indica timeout. |
| `spawnError` | `{ code?: string; message: string } \| undefined` | Error de creación del proceso. |
| `passed` | `boolean` | Solo true si `exitCode === 0`, sin timeout ni spawn error. |

## TaskOutcome

Estados por tarea:

- `completed`: prueba GREEN, marcador confirmado, incluye `testPath` y cantidad de intentos.
- `failed`: circuit breaker, RED no observado, test path inválido, fallo de marcado o error de test.
- `skipped`: tarea ya completada o flujo sin tareas.

Transiciones principales:

```text
pending -> red-verified -> implementing -> green -> marked-complete
pending -> skipped (si completed=true)
red-verified -> failed (red-not-observed, no command, timeout o maxRetries agotado)
green -> failed (marking error)
all tasks green + marked -> refactoring -> final-test
final-test -> completed | failed(regression)
```

## RunResult

| Campo | Tipo | Descripción |
|---|---|---|
| `status` | `completed \| failed \| no-tasks \| no-test-command` | Estado global. |
| `tasksPath` | `string` | Archivo de tareas resuelto. |
| `testCommand` | `TestCommand \| undefined` | Comando detectado. |
| `completedTasks` | `string[]` | IDs o descripciones confirmadas. |
| `failedTask` | `{ id: string; lastTest?: TestResult; testPath?: string; error: string } \| undefined` | Último fallo accionable. |
| `refactor` | `{ performed: boolean; finalTest?: TestResult }` | Resultado de fase final. |
