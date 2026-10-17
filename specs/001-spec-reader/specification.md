# 001-spec-reader — Specification

## Resumen

Crear los primeros dos nodos del agente. Estos nodos reciben la descripción de un proyecto de spec (carpeta del proyecto y nombres de archivo de plan y tasks), construyen las rutas absolutas a esos archivos y leen su contenido.

## Input

El agente recibe como entrada un objeto con tres campos:

- `path`: string, ruta de la carpeta del proyecto.
- `plan`: string, nombre relativo del archivo de plan.
- `tasks`: string, nombre relativo del archivo de tasks.

## Nodo 1 — `readInput`

- Toma las 3 entradas (`path`, `plan`, `tasks`).
- Valida que `path` sea un string no vacío.
- Construye el estado inicial guardando `path`, `plan` y `tasks`.

## Nodo 2 — `readFiles`

- Toma el estado producido por el nodo 1.
- Arma la ruta a los archivos usando `path.join(path, plan)` y `path.join(path, tasks)`.
- Lee el contenido de ambos archivos en UTF-8.
- Actualiza el estado con `planPath`, `tasksPath`, `planContent` y `tasksContent`.

## Estado

Se crea un estado (`SpecReaderState`) que guarda `path` para reutilizarlo entre nodos:

```
path: string
plan: string
tasks: string
planPath: string
tasksPath: string
planContent: string
tasksContent: string
```

## Output

El agente devuelve el estado completo, con las rutas resueltas y los contenidos leídos.