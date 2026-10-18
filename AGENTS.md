# langgraph-harness

Harness enfocado en proyectos **LangGraph.js** (TypeScript, ESM, Functional API).
Es el workspace raíz donde se desarrollan agentes con el flujo spec-kit
(`specify → clarify → plan → tasks → implement`).

## Estructura

- `.specify/` — configuración y memoria de spec-kit (constitución, templates, workflows, hooks git).
- `charlIA/` — agente de ejemplo, incluido como **git submodule** (repo `carloscba/charlIA`).
- `specs/` — specs del proyecto activo, con numeración secuencial (`001-`, `002-`, …).

## Convenciones LangGraph.js (aplican a cada agente)

- Functional API: `entrypoint`, `task`, `addMessages`; el agente vive en `src/index.ts`.
- Tools con `tool()` de `@langchain/core/tools` y schemas zod v4.
- ESM + `moduleResolution: NodeNext`: imports relativos con extensión `.js`.
- LLM vía **OpenRouter** con `@langchain/openai` (`baseURL = https://openrouter.ai/api/v1`).
- Secrets solo en `.env` (gitignored); `.env.example` es la plantilla.

## Comandos (dentro de cada agente)

- `npm run dev` — hot reload (`tsx watch src/index.ts`).
- `npm run start` — corre el agente una vez (llamada real al LLM).
- `npm run typecheck` — `tsc --noEmit`.

## Submódulo

```bash
git submodule update --init --recursive
```

<!-- SPECKIT START -->
For additional context about technologies to be used, project structure,
shell commands, and other important information, read the current plan:
`specs/005-tdd-implementing-agent/plan.md`
<!-- SPECKIT END -->
