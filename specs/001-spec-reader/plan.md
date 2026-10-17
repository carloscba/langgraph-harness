# 001-spec-reader — Plan

## Arquitectura

LangGraph **Functional API** (`entrypoint`, `task`). No se usa `StateGraph`.

El "estado" es un objeto tipado que se pasa de un task a otro dentro de la ejecución del entrypoint.

## Estado

```ts
interface SpecReaderState {
  path: string;
  plan: string;
  tasks: string;
  planPath: string;
  tasksPath: string;
  planContent: string;
  tasksContent: string;
}
```

## Nodos

### `readInput` (task)

- Firma: recibe `{ path: string; plan: string; tasks: string }`.
- Valida `path` no vacío (error claro si es vacío).
- Devuelve `{ path, plan, tasks }` como estado inicial.

### `readFiles` (task)

- Firma: recibe el estado (`SpecReaderState`).
- Construye `planPath = path.join(state.path, state.plan)` y `tasksPath = path.join(state.path, state.tasks)`.
- Lee ambos archivos con `fs/promises.readFile` en `"utf-8"`.
- Devuelve el estado completo con rutas y contenidos.

## Orquestación

```ts
const agent = entrypoint({ name: "agent" }, async (input) => {
  const initial = await readInput(input);
  const final = await readFiles(initial);
  return final;
});
```

## Archivos

- `src/nodes.ts` — estado y nodos.
- `src/index.ts` — entrypoint que orquesta los nodos.

## Verificación

- `npm run typecheck` (no hay tests ni lint en el proyecto).
- Prueba manual con un `invoke` sobre un proyecto con `plan.md` y `tasks.md`.