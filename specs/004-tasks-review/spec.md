# Feature Specification: Revisión de Tasks (eliminar lectura del plan)

**Feature Branch**: `004-tasks-review`

**Created**: 2026-10-17

**Status**: Draft

**Input**: User description: "Plan: eliminar lectura del plan y centrarse en path + tasks"

## Clarifications

### Session 2026-10-17

- Q: ¿Qué debe hacer el agente si el archivo de tasks está vacío (0 bytes)? → A: Informar que no hay contenido que revisar, sin llamar al LLM; devolver el estado con `tasksReview` descriptivo.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Revisar tasks con solo un path (Priority: P1)

Un desarrollador quiere obtener feedback sobre las tasks de una spec indicando únicamente la carpeta de la spec. Ya no necesita (ni puede) indicar un archivo de plan: el agente se enfoca exclusivamente en el archivo de tasks.

**Why this priority**: Es el cambio central del feature. Elimina una entrada redundante y simplifica el uso del agente a un solo argumento obligatorio (`path`).

**Independent Test**: Se ejecuta el agente indicando solo `path` y se verifica que lee el archivo de tasks por defecto (`tasks.md`), lo revisa y devuelve el feedback.

**Acceptance Scenarios**:

1. **Given** una carpeta de spec con un archivo `tasks.md`, **When** el desarrollador ejecuta el agente pasando solo `path`, **Then** el agente lee `tasks.md` y devuelve un estado con `tasksPath`, `tasksContent` y `tasksReview`.
2. **Given** una carpeta de spec sin archivo de plan, **When** el desarrollador ejecuta el agente pasando solo `path`, **Then** el agente completa el flujo sin error (ya no intenta leer ningún plan).

---

### User Story 2 - Elegir un nombre de archivo de tasks distinto (Priority: P2)

Un desarrollador puede indicar un nombre de archivo de tasks distinto al valor por defecto, para specs que usan otra convención de nombres.

**Why this priority**: Mantiene la flexibilidad actual sin reintroducir el plan. Es un caso de uso secundario pero necesario.

**Independent Test**: Se ejecuta el agente pasando `path` y un nombre de archivo de tasks explícito, y se verifica que lee ese archivo en lugar del default.

**Acceptance Scenarios**:

1. **Given** una carpeta de spec con un archivo de tasks de nombre no estándar, **When** el desarrollador ejecuta el agente pasando `path` y ese nombre, **Then** el agente lee ese archivo y devuelve el review correspondiente.
2. **Given** una carpeta de spec con `tasks.md`, **When** el desarrollador ejecuta el agente sin indicar nombre de tasks, **Then** se usa `tasks.md` como valor por defecto.

---

### User Story 3 - Mensaje de error claro ante path vacío (Priority: P3)

Un desarrollador que omite el argumento `path` recibe un mensaje de uso claro, sin que el agente falle de forma confusa.

**Why this priority**: Mejora la experiencia de uso y evita errores silenciosos, pero no aporta funcionalidad nueva.

**Independent Test**: Se ejecuta el agente sin `path` y se verifica que imprime un mensaje de uso y termina con código de error.

**Acceptance Scenarios**:

1. **Given** que no se proporciona `path` (o es un string vacío), **When** el desarrollador ejecuta el agente, **Then** se muestra un mensaje de uso claro y el proceso termina con un código distinto de cero.

---

### Edge Cases

- ¿Qué ocurre si el archivo de tasks indicado no existe en la carpeta? Se debe devolver un error legible (falla al leer el archivo).
- ¿Qué ocurre si `path` contiene solo espacios en blanco? Debe tratarse como vacío y rechazarse.
- ¿Qué ocurre si el archivo de tasks está vacío? El agente NO llama al LLM; devuelve el estado final con `tasksReview` = mensaje informando que no hay contenido que revisar.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE aceptar un argumento `path` obligatorio que indica la carpeta de la spec a revisar.
- **FR-002**: El sistema DEBE aceptar un argumento opcional con el nombre del archivo de tasks, cuyo valor por defecto es `tasks.md`.
- **FR-003**: El sistema NO DEBE requerir ni leer ningún archivo de plan.
- **FR-004**: El sistema DEBE resolver la ruta completa del archivo de tasks a partir de `path` y el nombre de tasks.
- **FR-005**: El sistema DEBE leer el contenido del archivo de tasks y dejarlo disponible en el estado.
- **FR-006**: El sistema DEBE enviar el contenido de tasks a revisión y almacenar el feedback devuelto en el estado.
- **FR-007**: El sistema DEBE rechazar un `path` vacío o solo de espacios con un mensaje de error claro.
- **FR-008**: El sistema DEBE devolver un estado final que incluya `path`, el nombre de tasks, la ruta resuelta de tasks, el contenido de tasks y el review de tasks.

### Key Entities *(include if feature involves data)*

- **Archivo de tasks**: contenido de texto con las tareas de una spec; es la única entrada de contenido que el agente lee y revisa.
- **Review de tasks**: feedback textual generado por la revisión sobre el contenido de tasks (claridad, completitud, riesgos, pasos faltantes).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un desarrollador obtiene un review del archivo de tasks indicando únicamente `path` (sin argumentos de plan), completando el flujo de principio a fin.
- **SC-002**: El agente ya no expone, requiere ni menciona "plan" en su interfaz de uso, entrada, estado ni prompt de revisión.
- **SC-003**: El flujo de revisión de tasks completa exitosamente en los casos de prueba manuales definidos (path solo, y path con nombre de tasks explícito).
- **SC-004**: La verificación de tipos del proyecto concluye sin errores tras los cambios.

## Assumptions

- El archivo de plan ya no es necesario para la revisión y puede eliminarse del flujo sin perder valor.
- El nombre por defecto del archivo de tasks es `tasks.md`.
- El formato de feedback (claridad, completitud, riesgos y pasos faltantes) se mantiene igual al de la revisión previa, aplicado ahora a las tasks.
- Los comandos de verificación (typecheck y prueba manual) se ejecutan dentro del proyecto del agente (submódulo `charlIA`).
