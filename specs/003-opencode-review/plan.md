# 003-opencode-review — Plan

## Arquitectura

LangGraph **Functional API** (`entrypoint`, `task`). Se agrega un tercer task después de `readFiles`.

La invocación al CLI de OpenCode se encapsula en un helper dedicado (`src/opencode.ts`) para no mezclar la lógica de spawn/export con los tasks.

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
  planReview?: string;
}
```

## Nodos

### `reviewPlan` (task)

- Firma: recibe `SpecReaderState`.
- Arma un prompt a partir de `planContent`, con la mención `@explore`.
- Llama a `runOpenCodeReview({ planContent, cwd: state.path })`.
- Devuelve el estado con `planReview` agregado.

## Helper CLI

### `runOpenCodeReview` en `src/opencode.ts`

- Construye el prompt:
  ```
  @explore Revisá el siguiente plan de una spec y dame un feedback detallado sobre claridad, completitud, riesgos y posibles pasos faltantes:
  <planContent>
  Respondé en español.
  ```
- Ejecuta:
  ```bash
  opencode run --dangerously-skip-permissions --format json --model opencode-go/deepseek-v4-pro <prompt>
  ```
- Pasa la variable de entorno `OPENCODE_PERMISSION` con reglas de solo lectura:
  - `allow`: `read`, `glob`, `grep`, `list`, `lsp`, `skill`, `webfetch`, `websearch`
  - `deny`: `edit`, `bash`, `task`, `external_directory`, `question`, `doom_loop`
- Usa `spawn` de `node:child_process` con:
  - `cwd` = `state.path`
  - `stdio: ["ignore", "pipe", "pipe"]`
  - timeout de 5 minutos.
- Del primer evento JSON (`step_start`) obtiene `sessionID`.
- Ejecuta `opencode export <sessionID>` y parsea el JSON de stdout.
- Extrae el texto del último mensaje del asistente (`role === "assistant"`, parts de tipo `text`).
- Rechaza con error claro si:
  - el binario no existe (`ENOENT`),
  - `opencode run` o `opencode export` terminan con código distinto de 0,
  - no se encuentra `sessionID` o texto de asistente,
  - se excede el timeout.

## Orquestación

```ts
const agent = entrypoint({ name: "agent" }, async (input) => {
  const initial = await readInput(input);
  const withFiles = await readFiles(initial);
  const reviewed = await reviewPlan(withFiles);
  return reviewed;
});
```

## Archivos

- `src/opencode.ts` — helper para ejecutar `opencode run` + `opencode export`.
- `src/nodes.ts` — estado y nodos (agrega `reviewPlan`).
- `src/index.ts` — entrypoint que orquesta los tres nodos.

## Nota sobre `@explore`

El flag `--agent explore` no puede usarse directamente porque `explore` es un subagente; el CLI cae al agente default. Por eso se incluye `@explore` en el prompt y se recupera la respuesta vía exportación de sesión.

## Verificación

- `npm run typecheck`.
- Prueba manual: `npm run start -- ./specs/002-spec-invocation`.
