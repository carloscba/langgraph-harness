# 003-opencode-review — Specification

## Resumen

Agregar un tercer nodo que envíe el contenido del plan (`planContent`) al CLI de OpenCode (`opencode run`) usando el agente `@explore`, el modelo `opencode-go/deepseek-v4-pro`, y guarde el feedback devuelto en el estado.

## Input

El nodo recibe el estado completo producido por `readFiles`:

- `path`: string
- `plan`: string
- `tasks`: string
- `planPath`: string
- `tasksPath`: string
- `planContent`: string
- `tasksContent`: string

## Nodo 3 — `reviewPlan`

- Recibe `SpecReaderState`.
- Construye un prompt en español que menciona a `@explore` y le pide revisar el plan.
- Ejecuta `opencode run --dangerously-skip-permissions --format json --model opencode-go/deepseek-v4-pro <prompt>` desde la carpeta del proyecto (`path`).
- Configura `OPENCODE_PERMISSION` para permitir solo lectura (`read`, `glob`, `grep`, `list`, etc.) y denegar escritura, bash, task y otras operaciones destructivas.
- Obtiene el `sessionID` del primer evento JSON.
- Ejecuta `opencode export <sessionID>` y extrae el texto del último mensaje del asistente.
- Agrega el campo `planReview` al estado.

## Estado

Se extiende `SpecReaderState` con:

```
planReview?: string
```

(`planReview` es opcional hasta que el nodo 3 lo complete.)

## Output

El agente devuelve el estado completo, incluyendo `planReview` con el feedback del agente.

## Uso

```bash
npm run start -- ./specs/001-spec-reader plan.md tasks.md
```
