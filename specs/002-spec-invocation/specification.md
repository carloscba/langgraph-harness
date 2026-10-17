# 002-spec-invocation — Specification

## Resumen

Agregar la invocación del `entrypoint` en `src/index.ts` para ejecutar el flujo con `npm run start`, tomando el input desde argumentos de línea de comandos.

## Input

El agente recibe el input parseando `process.argv`:

- `path`: `process.argv[2]`, string, ruta de la carpeta del proyecto (obligatorio).
- `plan`: `process.argv[3]`, string, nombre relativo del archivo de plan (default `"plan.md"`).
- `tasks`: `process.argv[4]`, string, nombre relativo del archivo de tasks (default `"tasks.md"`).

## Comportamiento

1. Parsear los argumentos CLI.
2. Validar que `path` esté presente y no sea un string vacío.
3. Invocar el `entrypoint` con el input construido.
4. Imprimir el resultado (`SpecReaderState`) como JSON en consola.

## Output

El estado completo devuelto por el flujo, incluyendo rutas resueltas y contenidos leídos.

## Uso

```bash
npm run start -- ./specs/001-spec-reader plan.md tasks.md
```

Si `plan` y `tasks` usan los nombres por defecto:

```bash
npm run start -- /home/carloscba/Develop/charlIA specs/002-spec-invocation/plan.md specs/002-spec-invocation/tasks.md
```
