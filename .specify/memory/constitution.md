<!--
Sync Impact Report
==================
Version change: (template) → 1.0.0
Modified principles: none (first ratification)
Added sections: Development Workflow, Technology Constraints
Removed sections: none
Templates requiring updates:
  - .specify/templates/plan-template.md      ✅ aligned (Constitution Check gate referenced)
  - .specify/templates/spec-template.md      ✅ aligned (no principle-specific sections)
  - .specify/templates/tasks-template.md     ✅ aligned (typecheck gate noted in Phase 1)
  - .specify/templates/constitution-template.md ⚠ unchanged (kept as canonical template)
Follow-up TODOs: none
-->

# langgraph-harness Constitution

## Core Principles

### I. LangGraph.js Functional API

Todo agente en este harness MUST construirse con LangGraph.js (TypeScript, ESM)
usando la **Functional API**: `entrypoint`, `task` y `addMessages`. El punto de
entrada del agente vive en `src/index.ts`. Las tools se definen con `tool()` de
`@langchain/core/tools` y schemas zod v4. Queda prohibido reintroducir la
StateGraph/`graph` API clásica sin justificación documentada.

### II. Spec-Driven Development (Spec Kit)

Toda feature MUST seguir el flujo spec-kit: `specify → clarify → plan → tasks →
implement`. Los specs viven en `specs/` con numeración secuencial (`001-`, `002-`,
…), y cada carpeta contiene `specification.md`, `plan.md` y `tasks.md`. Un spec
MUST ser revisado y verificado antes de ejecutar sus tareas; crear un spec nunca
implica implementar.

### III. ESM + NodeNext Module Resolution

El proyecto MUST usar ESM con `moduleResolution: NodeNext`. Los imports relativos
MUST llevar la extensión `.js` aunque el archivo fuente sea `.ts`
(`import ... from "./model.js"`). No se admiten `require`/CommonJS ni imports sin
extensión en rutas relativas.

### IV. OpenRouter-Only LLM Access

El modelo se accede EXCLUSIVAMENTE vía OpenRouter usando `@langchain/openai` con
`configuration.baseURL = https://openrouter.ai/api/v1` en `src/model.ts`. Están
prohibidos los clientes nativos de Anthropic, DeepSeek u OpenAI. Las credenciales
(`OPENROUTER_API_KEY`, `MODEL`) vienen de `.env`; para cambiar de modelo se edita
`MODEL` en `.env`, nunca el código.

### V. Secrets & Environment Hygiene (NON-NEGOTIABLE)

Las API keys reales solo pueden existir en `.env` (gitignored). `.env.example` es
la plantilla y NUNCA debe contener keys reales. Queda prohibido commitear o loguear
secrets. Cualquier PR/commit que filtre una key se revierte de inmediato.

### VI. Typecheck Gate & Simplicity

No se requieren pasos de build, lint ni tests: `tsx` ejecuta `.ts` directamente.
`npm run typecheck` (`tsc --noEmit`) MUST pasar antes de dar una feature por
completa. Aplicar YAGNI: empezar simple y no introducir abstracciones sin un
requisito concreto que las justifique.

## Technology Constraints

- **Lenguaje/Plataforma**: TypeScript sobre Node.js, ESM (`"type": "module"`).
- **Runtime de desarrollo**: `tsx` (watch para `dev`, directo para `start`).
- **Dependencias núcleo**: `@langchain/core`, `@langchain/langgraph`,
  `@langchain/openai`, `zod` (v4), `dotenv`.
- **Storage**: ninguno por defecto; los agentes son stateless salvo que un spec lo
  especifique.
- **Comandos del proyecto**: `npm run dev`, `npm run start`, `npm run typecheck`.

## Development Workflow

1. **Spec primero**: describir la feature con `/speckit.specify`; la spec resultante
   se revisa (no se implementa) y se resuelven clarificaciones con
   `/speckit.clarify`.
2. **Plan y tareas**: `/speckit.plan` produce `plan.md`; `/speckit.tasks` produce
   `tasks.md`. Ambos se verifican antes de implementar.
3. **Implementación**: `/speckit.implement` ejecuta las tareas por user story en
   orden de prioridad (P1 → P2 → …), validando cada story de forma independiente.
4. **Quality gate**: `npm run typecheck` limpio y ausencia de secrets commiteados
   antes de considerar una feature terminada.

## Governance

Esta constitución tiene prioridad sobre cualquier otra práctica o convención del
proyecto. Las enmiendas requieren documentar el cambio, justificar la complejidad
y actualizar los templates dependientes (plan, spec, tasks) en el mismo commit.
Todo PR/review debe verificar cumplimiento de los principios. Usar `AGENTS.md`
como guía de desarrollo en tiempo de ejecución.

**Version**: 1.0.0 | **Ratified**: 2026-10-17 | **Last Amended**: 2026-10-17
