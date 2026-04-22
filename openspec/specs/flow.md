# Flow Specifications

Este documento describe los flujos de usuario y las transiciones de estado del sistema Advisory Board.

## Flujos Principales de Usuario

### 1. Autenticación
**Propósito**: Controlar el acceso al demo.
- **Acción**: El usuario introduce la contraseña en la pantalla de login.
- **Resultado**: Se genera un token UUID almacenado en memoria (`POST /api/auth/login`).
- **Estado**: Sesión activa. Token guardado en el cliente.

---

### 2. Carga del Board
**Propósito**: Mostrar el panel de expertos antes de cualquier consulta.
- **Acción**: El usuario accede a `/[boardId]`.
- **Resultado**: La app carga la configuración del board y los agentes (`GET /api/[boardId]/agents`).
- **Estado**: Board en `idle`. Todos los agentes visibles con avatar, nombre y cargo. Input de pregunta activo. Preguntas de ejemplo visibles.

---

### 3. Envío de Consulta
**Propósito**: Iniciar una evaluación del board.
- **Acción**: El usuario escribe una pregunta o selecciona un ejemplo y pulsa enviar.
- **Resultado**: Se abre un stream SSE (`POST /api/[boardId]/query`). El input se desactiva.
- **Estado**: Board en `querying`. Orquestador clasifica el tipo de query.

---

### 4. Respuesta en Paralelo (Ronda 1)
**Propósito**: Los agentes evalúan la pregunta desde su perspectiva.
- **Proceso**: Los agentes responden en paralelo (batches configurables para respetar rate limits).
- **Por cada agente**:
  - Estado: `thinking` → aparece animación en la tarjeta.
  - A medida que llega el stream: los campos estructurados (recommendation, primary_concern, path_forward) aparecen en la tarjeta.
  - Estado: `done` → el badge de veredicto queda fijo.
- **Consensus Bar**: se actualiza en tiempo real con cada agente que termina.
- **Estado del board**: `round_1_in_progress` → `round_1_done`.

---

### 5. Reconsideración (Opcional)
**Propósito**: El orquestador identifica conflictos relevantes y pide a agentes específicos que revisen su posición.
- **Condición**: El orquestador detecta conflictos o incoherencias significativas entre veredictos.
- **Proceso**:
  - Aparece mensaje en la UI: *"The orchestrator has asked [Agent X] and [Agent Y] to reconsider their position."*
  - Los agentes afectados pasan a estado `reconsidering`.
  - Sus veredictos pueden cambiar — el badge anterior queda visible como referencia.
- **Estado del board**: `reconsidering` → `reconsideration_done`.

---

### 6. Síntesis del Orquestador
**Propósito**: Producir el Decision Brief consolidado.
- **Proceso**: El orquestador genera un resumen con los veredictos de todos los agentes.
- **Resultado**:
  - Resumen ejecutivo visible directamente (2–4 líneas).
  - Botón "View full analysis" que expande el Decision Brief completo en markdown.
  - Consensus Bar muestra el estado final con distribución de veredictos.
- **Estado del board**: `synthesizing` → `done`.

---

### 7. Exploración de Expertos
**Propósito**: El usuario profundiza en la opinión de un experto específico.
- **Ver respuesta completa**: Clic en la tarjeta del agente muestra su respuesta completa en markdown.
- **Chat 1:1**: Botón en la tarjeta abre un modal de chat con el agente.
  - El agente conoce su posición en el board y el contexto de la sesión.
  - El historial del chat dura mientras dure la sesión del navegador.
  - Respuestas en streaming (`POST /api/[boardId]/expert/[agentId]/chat`).

---

### 8. Nueva Consulta
**Propósito**: El usuario envía otra pregunta al mismo board.
- **Acción**: El usuario escribe una nueva pregunta con el board en estado `done`.
- **Resultado**: La consulta anterior se archiva en el historial de la sesión (visible en la UI). Comienza un nuevo ciclo desde el paso 3.
- **Estado**: El historial muestra cada query con su Consensus Bar y resumen.

---

## Transiciones de Estado del Board

```
idle
  ↓  (usuario envía pregunta)
querying
  ↓  (orquestador clasifica query_type)
round_1_in_progress
  ↓  (todos los agentes terminan)
round_1_done
  ↓  (si hay conflictos detectados)
reconsidering               ← opcional
  ↓  (agentes afectados terminan)
reconsideration_done        ← opcional
  ↓
synthesizing
  ↓  (Decision Brief generado)
done
  ↓  (usuario envía nueva pregunta)
querying  ← vuelve al ciclo
```

## Transiciones de Estado de un Agente

```
idle
  ↓  (board entra en querying)
thinking
  ↓  (primera respuesta llega del stream)
streaming
  ↓  (stream completa)
done
  ↓  (si orquestador pide reconsideración)
reconsidering               ← opcional
  ↓
done
```
