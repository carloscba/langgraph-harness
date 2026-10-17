# 002-spec-invocation — Plan

## Arquitectura

LangGraph **Functional API** (`entrypoint`, `task`). No se modifica `StateGraph` ni los nodos existentes.

La invocación del agente vive en `src/index.ts`, junto con la lógica de parseo de argumentos CLI.

## Pasos

1. Parsear `process.argv` en `src/index.ts` para obtener `path`, `plan` y `tasks`.
2. Asignar defaults: `plan = "plan.md"` y `tasks = "tasks.md"`.
3. Validar que `path` esté presente y no sea vacío; lanzar error claro si falta.
4. Invocar el `entrypoint` con el input construido.
5. Imprimir el resultado como JSON en consola.

## Archivos

- `src/index.ts` — parseo de argumentos e invocación del agente.

## Verificación

- `npm run typecheck`.
- Prueba manual: `npm run start -- ./specs/001-spec-reader`.
