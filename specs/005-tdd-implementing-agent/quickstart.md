# Quickstart: Agente implementador TDD

## Prerequisites

- Node.js y npm disponibles.
- Dependencias instaladas en `charlIA/`.
- CLI `opencode` disponible en `PATH` y autenticado para los modelos `opencode-go`.
- Un repositorio objetivo con `package.json` y un script `test`, `vitest` o `jest`.
- Un archivo de tareas con al menos una tarea pendiente y un formato que `opencode` pueda interpretar.

## Typecheck del agente

```bash
cd charlIA
npm install
npm run typecheck
```

Resultado esperado: TypeScript termina sin errores.

## Ejecución básica

```bash
npm run start -- /absolute/path/to/project
```

Usa `/absolute/path/to/project/tasks.md` y `maxRetries=5`.

## Archivo alternativo y retries

```bash
npm run start -- /absolute/path/to/project work-items.md 2
```

El agente debe:

1. Leer y normalizar las tareas mediante `opencode` sin asumir checkboxes.
2. Detectar el comando desde `package.json`.
3. Escribir una prueba para la primera tarea pendiente.
4. Ejecutarla y observar RED desde el código de salida.
5. Implementar/fijar hasta dos veces, usando el mismo `testPath`.
6. Marcar solo la tarea GREEN y continuar en orden.
7. Ejecutar una única refactorización final cuando todas las tareas estén GREEN.
8. Ejecutar la prueba final y emitir JSON.

## Validaciones de fallo

### Sin path

```bash
npm run start --
```

Esperado: mensaje de uso en stderr y código distinto de cero.

### Sin comando de test

Usar un proyecto cuyo `package.json` no tenga `scripts.test`, `scripts.vitest` ni `scripts.jest`.

Esperado: `status: "no-test-command"`; no se llama a Test Writer ni se escriben archivos.

### RED no observado

Usar una tarea cuya prueba generada ya pase.

Esperado: `status: "failed"`, error `red-not-observed`; no se llama al implementador ni se marca la tarea.

### Circuit breaker

```bash
npm run start -- /absolute/path/to/project tasks.md 0
```

Esperado: se verifica RED y el flujo se detiene sin intento de implementación.

### Regresión en refactor

Preparar un refactor que haga fallar la corrida final.

Esperado: `status: "failed"` con el `TestResult` final; no se continúa con ninguna operación posterior.

## Acceptance checklist

- [ ] Procesa múltiples tareas en orden.
- [ ] Funciona con un formato de tareas que no usa checkboxes spec-kit.
- [ ] La ejecución dirigida devuelve RED antes del primer fix.
- [ ] `stderr` exacto llega al implementador.
- [ ] Nunca marca una tarea antes de GREEN y confirmación del marker.
- [ ] Detiene el flujo al agotar `maxRetries`.
- [ ] Refactoriza una sola vez al final.
- [ ] La corrida final pasa o reporta regresión.
- [ ] No crea commits.
