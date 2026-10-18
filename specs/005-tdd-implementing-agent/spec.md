# Feature Specification: Agente implementador TDD determinista (Spec-Driven)

**Feature Branch**: `005-tdd-implementing-agent`

**Created**: 2026-10-17

**Status**: Draft

**Input**: User description: "Reemplazar el flujo actual de revisión de tasks del agente charlIA por un agente implementador basado en TDD determinista (Spec-Driven). Dado el path del proyecto y el archivo tasks.md, el agente parsea las tareas y, por cada una, ejecuta el bucle Red-Green-Refactor: escribir una prueba (RED), correrla con el test runner del repo, implementar/arreglar el código mínimo (GREEN) hasta que la prueba pase o se agote un límite de reintentos, y luego refactorizar sin cambiar el comportamiento verificado. Al pasar, marca la tarea como completada en tasks.md y pasa a la siguiente. MVP: flujo completo Red-Green-Refactor."

## Clarifications

### Session 2026-10-17

- Q: ¿El refactor se ejecuta por cada tarea o una única vez al final? → A: Una única fase de refactor al final del flujo, después de procesar todas las tareas, sobre el conjunto de cambios.
- Q: ¿Cuántas pruebas puede generar una tarea? → A: Una prueba por tarea.
- Q: ¿El archivo de tareas tiene un formato fijo? → A: No; es agnóstico al formato, y el parseo y el marcado de completadas se delegan al CLI opencode para adaptarse a cualquier formato.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ejecutar el ciclo Red-Green sobre una tarea (Priority: P1)

Un desarrollador entrega al agente el path de un repositorio y un archivo `tasks.md`. Para una tarea concreta, el agente escribe una prueba que aún no pasa (RED), la ejecuta para confirmar el fallo e implementa o corrige el código mínimo necesario hasta que esa prueba pase (GREEN).

**Why this priority**: Es el núcleo del feature. Sustituye por completo la revisión pasiva por un ciclo activo de implementación guiada por pruebas. Sin este ciclo, ninguna tarea puede completarse.

**Independent Test**: Se ejecuta el agente con un repositorio y un `tasks.md` que contiene una única tarea; al terminar, existe un archivo de prueba que fallaba antes de implementar y que ahora pasa, y la tarea queda marcada como completada.

**Acceptance Scenarios**:

1. **Given** un repositorio con un `tasks.md` que contiene una tarea, **When** el agente procesa esa tarea, **Then** escribe un archivo de prueba para ella y devuelve su ruta para ejecutarla de forma dirigida.
2. **Given** la prueba recién escrita, **When** el agente la ejecuta con el comando de test del repositorio, **Then** la ejecución falla (RED) y el agente registra el código de salida y el error exacto.
3. **Given** el fallo de la prueba, **When** el agente implementa o corrige el código mínimo, **Then** la misma prueba pasa (GREEN) al re-ejecutarse.

---

### User Story 2 - Procesar múltiples tareas en secuencia (Priority: P1)

Un `tasks.md` típico contiene varias tareas. El agente las parsea y las procesa una por una, en orden, marcando cada tarea como completada en el propio `tasks.md` únicamente cuando su prueba pasa, y continuando con la siguiente.

**Why this priority**: Es el flujo real de trabajo: un archivo de tareas rara vez tiene una sola entrada. Junto con US1 forma el MVP completo.

**Independent Test**: Se ejecuta el agente sobre un `tasks.md` con al menos dos tareas; al finalizar, ambas quedan marcadas como completadas y cada una tiene su prueba en verde.

**Acceptance Scenarios**:

1. **Given** un `tasks.md` con varias tareas, **When** el agente parsea el archivo, **Then** identifica la lista completa de tareas y su estado de completitud.
2. **Given** varias tareas, **When** una tarea termina en verde, **Then** el agente la marca como completada en `tasks.md` y pasa a la siguiente en orden.
3. **Given** que una tarea intermedia falla y se agota su límite de reintentos, **When** el agente lo detecta, **Then** detiene el flujo y no procesa las tareas restantes.

---

### User Story 3 - Refactor único del conjunto de cambios (Priority: P2)

Una vez que todas las tareas han quedado en verde, el agente ejecuta una única pasada de refactor sobre el conjunto de código generado: revisa estilo, tipos y simplicidad, sin alterar el comportamiento que las pruebas verifican, y ejecuta los tests una última vez para descartar regresiones.

**Why this priority**: Da coherencia al conjunto de cambios acumulado. Refactorizar por tarea sería redundante y perdería la visión global; hacerlo una sola vez al final mantiene el flujo simple y evita regresiones acumuladas.

**Independent Test**: Se ejecuta el agente sobre un `tasks.md` con varias tareas; al terminar todas en verde, el agente realiza una única pasada de refactor y la corrida final de tests concluye en verde.

**Acceptance Scenarios**:

1. **Given** todas las tareas completadas en verde, **When** el agente ejecuta la fase de refactor, **Then** revisa estilo, tipos y simplicidad del código generado sin alterar el comportamiento verificado.
2. **Given** el refactor aplicado, **When** el agente ejecuta los tests una última vez, **Then** la ejecución concluye en verde, descartando regresiones.
3. **Given** que no hubo cambios que refactorizar (todas las tareas ya estaban completas antes de iniciar), **When** el agente llega a la fase de refactor, **Then** la omite y reporta que no hubo cambios.

---

### User Story 4 - Circuit breaker ante reintentos agotados (Priority: P2)

Si una tarea no logra pasar su prueba tras un número máximo de reintentos configurable (por defecto 5), el agente se detiene y reporta el último error para intervención humana, en lugar de reintentar indefinidamente.

**Why this priority**: Protege contra bucles infinitos y costos descontrolados. Sin él, el agente podría quedar atascado en una tarea irresoluble.

**Independent Test**: Se ejecuta el agente sobre una tarea diseñada para no pasar nunca (o con `maxRetries` muy bajo); al agotarse el límite, el agente termina reportando el último error sin reintentar más allá del límite.

**Acceptance Scenarios**:

1. **Given** una tarea cuya prueba sigue fallando, **When** se alcanza el máximo de reintentos, **Then** el agente se detiene y reporta el último error y el contexto del fallo.
2. **Given** un valor de `maxRetries` proporcionado por el desarrollador, **When** el agente ejecuta el ciclo, **Then** usa ese valor como límite; si no se proporciona, usa 5.
3. **Given** que el flujo se detuvo por reintentos agotados, **When** el desarrollador revisa la salida, **Then** dispone de la prueba, el archivo afectado y el error exacto para intervenir manualmente.

---

### User Story 5 - Interfaz de línea de comandos (Priority: P3)

El desarrollador invoca el agente pasando un `path` obligatorio, un nombre de archivo de tareas opcional (por defecto `tasks.md`) y, opcionalmente, un `maxRetries`.

**Why this priority**: Define cómo se usa el agente y sus límites configurables, pero no añade lógica de implementación nueva.

**Independent Test**: Se invoca el agente con solo `path` y con `path` + `maxRetries`; en ambos casos se resuelven los defaults esperados (`tasks.md` y 5, respectivamente).

**Acceptance Scenarios**:

1. **Given** una invocación sin `path`, **When** el desarrollador ejecuta el agente, **Then** muestra un mensaje de uso claro y termina con código distinto de cero.
2. **Given** una invocación con `path` y sin nombre de tareas, **When** el agente se ejecuta, **Then** usa `tasks.md` como archivo de tareas por defecto.
3. **Given** una invocación con `path` y sin `maxRetries`, **When** el agente se ejecuta, **Then** usa 5 reintentos como límite por defecto.

---

### Edge Cases

- ¿Qué ocurre si `tasks.md` no existe en el path indicado? El agente debe devolver un error legible que incluya la ruta resuelta.
- ¿Qué ocurre si `tasks.md` está vacío o no contiene tareas? El agente no debe llamar a ningún modelo ni escribir nada; devuelve un mensaje informativo.
- ¿Qué ocurre si el repositorio no define ningún comando de test detectable? El agente debe reportar que no pudo determinar el comando de test y detenerse sin escribir implementación.
- ¿Qué ocurre si la prueba escrita ya pasa en la primera ejecución (no hay RED)? El agente debe detectar esta anomalía y reportarla, en lugar de dar por válido un ciclo sin verificación.
- ¿Qué ocurre si el comando de test no existe o expira por tiempo? El agente debe capturar el error y tratarlo como un fallo de la prueba para el reintento correspondiente.
- ¿Qué ocurre si todas las tareas ya estaban completas al iniciar? No hay código nuevo que refactorizar; el agente omite la fase de refactor y reporta que no hubo cambios.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE aceptar un argumento `path` obligatorio que identifica el repositorio objetivo.
- **FR-002**: El sistema DEBE aceptar un argumento opcional con el nombre del archivo de tareas (por defecto `tasks.md`).
- **FR-003**: El sistema DEBE aceptar un argumento opcional `maxRetries` (por defecto 5) que limita los reintentos del ciclo de implementación.
- **FR-004**: El sistema DEBE parsear el archivo de tareas de forma agnóstica al formato, delegando en el CLI `opencode`, y extraer la lista ordenada de tareas con su estado de completitud.
- **FR-005**: El sistema DEBE procesar cada tarea ejecutando el ciclo Red-Green: escribir una prueba, ejecutarla (debe fallar), e implementar/arreglar el código mínimo hasta que pase.
- **FR-006**: El sistema DEBE determinar automáticamente el comando de test del repositorio objetivo a partir de su `package.json` (prefiriendo `test`, y en su defecto `vitest`/`jest`).
- **FR-007**: El sistema DEBE ejecutar el comando de test mediante un subproceso y capturar el código de salida, la salida estándar y la salida de error; la salida de la terminal es la única fuente de verdad sobre si una prueba pasa o falla.
- **FR-008**: El nodo que escribe la prueba DEBE devolver la ruta del archivo de prueba para que la ejecución de tests pueda apuntar específicamente a ese archivo.
- **FR-009**: El nodo que implementa/arregla el código DEBE recibir únicamente el contexto mínimo: la prueba, el archivo afectado y el error exacto del fallo (no el repositorio completo).
- **FR-010**: Los nodos que escriben o interpretan archivos (parseo de tareas, escritura de prueba, implementación/arreglo, refactor y marcado de completadas) DEBEN delegar en el CLI `opencode` con permisos de escritura restringidos al repositorio objetivo.
- **FR-011**: El nodo implementador DEBE usar el modelo `Qwen3.8 flash` y el nodo de refactor DEBE usar el modelo `mimo-v2.6 flash` (modelos de opencode-go).
- **FR-012**: El sistema DEBE ejecutar una única fase de refactor al final del flujo, una vez completadas todas las tareas, que revise estilo, tipos y simplicidad del código generado sin alterar el comportamiento que las pruebas verifican, y DEBE ejecutar los tests una última vez para descartar regresiones.
- **FR-013**: El sistema DEBE marcar una tarea como completada en `tasks.md` únicamente cuando su prueba pasa, preservando el formato original del archivo.
- **FR-014**: El sistema NO DEBE crear commits de git; su único efecto sobre el control de versiones debe limitarse a los cambios en archivos de código/prueba y el marcado en `tasks.md`.
- **FR-015**: El sistema DEBE detener el flujo y reportar el último error cuando una tarea agota `maxRetries`, para intervención humana.
- **FR-016**: El sistema DEBE reportar un resumen final indicando qué tareas se completaron y cuáles requieren intervención manual.

### Key Entities *(include if feature involves data)*

- **Tarea**: entrada individual del archivo de tareas, con una descripción y un estado (pendiente/completada); es la unidad de trabajo del ciclo.
- **Archivo de prueba**: archivo que codifica una expectativa verificable de una tarea; su ejecución determina RED o GREEN.
- **Resultado de test**: resultado de una ejecución del comando de test (código de salida, salida estándar y salida de error); es la fuente de verdad del ciclo.
- **Resumen de ejecución**: reporte final con el estado de cada tarea (completada o con error pendiente de intervención).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El agente procesa el 100 % de las tareas parseadas y marca cada una como completada únicamente después de que su prueba pasa.
- **SC-002**: Para cada tarea, la prueba falla antes de implementar (RED) y pasa después de implementar (GREEN), confirmado por la salida del comando de test.
- **SC-003**: Cuando una tarea agota el máximo de reintentos, el agente se detiene y reporta el último error en lugar de reintentar indefinidamente.
- **SC-004**: El refactor no introduce regresiones: la corrida final de tests posterior a la fase de refactor concluye en verde.
- **SC-005**: La verificación de tipos del agente concluye sin errores tras la implementación.

## Assumptions

- El archivo de tareas es agnóstico al formato: el agente no asume una sintaxis concreta (checkboxes de spec-kit, listas, etc.); la detección de tareas y su marcado como completadas se delegan al CLI `opencode` para adaptarse al formato del archivo de entrada.
- Cada tarea se implementa con una única prueba dirigida (una prueba por tarea); el flujo no genera suites de pruebas completas en el MVP.
- El CLI `opencode` está instalado y disponible en el PATH, y el repositorio objetivo es un proyecto Node.js con `package.json`.
- El agente delega los nodos que escriben/interpretan archivos al CLI `opencode` de forma intencional (pilar de diseño), para aprovechar su versatilidad de desarrollo y disponer de los modelos de opencode-go.
- El agente vive en el submódulo `charlIA` y reemplaza el flujo de revisión actual (`src/index.ts`, `src/nodes.ts`, `src/opencode.ts`).
- Las validaciones se ejecutan dentro del repositorio del agente; las pruebas TDD se ejecutan dentro del repositorio objetivo indicado por `path`.
