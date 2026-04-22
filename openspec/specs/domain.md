# Domain Specifications

Este documento describe las entidades de negocio y las reglas que rigen el sistema Advisory Board.

## Entidades Principales

### 1. Board
Representa un panel de expertos sintéticos configurado para un caso de uso específico.

**Atributos:**
- `id`: Identificador único. Determina la URL (`/[boardId]`).
- `name`: Nombre visible del board (ej. "Veterinary Advisory Board").
- `description`: Descripción breve del propósito del board.
- `example_questions`: Lista de preguntas de ejemplo precargadas en la UI.
- `consistency_check`: Boolean. Si los agentes pasan por verificación de claims tras responder. Default: `true`.

**Configuración:**
Cada board vive en `boards/[id]/board.yaml`. La app escanea esta carpeta al arrancar. Añadir un board nuevo es crear una carpeta — sin tocar código.

---

### 2. Agent
Representa un experto sintético dentro de un board.

**Atributos:**
- `id`: Identificador único dentro del board.
- `name`: Nombre del personaje (ej. "Dr. Sarah Chen").
- `credentials`: Titulaciones visibles en UI (ej. "DVM, DACVIM").
- `seat`: Posición en el board (ej. "Companion Animals").
- `specialty`: Área de foco (ej. "Dogs & cats clinical practice").
- `avatar`: Ruta a imagen de avatar.
- `prompt_file`: Ruta al archivo `.md` con el system prompt del agente.

**Conocimiento:**
Cada agente puede tener documentos de contexto en `boards/[boardId]/knowledge/[agentId]/`. El sistema RAG los indexa y los inyecta como contexto relevante en cada consulta.

---

### 3. Verdict
Resultado estructurado producido por un agente tras evaluar una pregunta.

**Atributos comunes:**
- `recommendation`: Valor según el tipo de query (ver prompt-architecture.md).
- `primary_concern`: Preocupación principal en una línea.
- `path_forward`: Camino recomendado en una línea.

**Atributos opcionales (según tipo):**
- `blocking_issue`: Problema que impide un endorsement completo.
- `condition`: Condición bajo la que endorsaría.
- `strength`: Argumento más fuerte a favor.
- `gap`: Qué falta para un veredicto positivo completo.

---

### 4. ConsensusScore
Distribución agregada de los veredictos de todos los agentes de un board para una query.

**Atributos:**
- `distribution`: Mapa de `recommendation_value → count`.
- `total`: Número total de agentes que respondieron.

Se calcula al finalizar la Ronda 1 y se actualiza si hay reconsideración.

---

### 5. BoardSession
Estado en memoria de una sesión de uso del board. No se persiste — se pierde al reiniciar.

**Atributos:**
- `board_id`: Board activo.
- `question`: Pregunta enviada por el usuario.
- `query_type`: Tipo clasificado por el orquestador (`evaluation` | `landscape` | `advisory`).
- `agent_states`: Estado de cada agente (streaming, verdict, full_text, reconsidering).
- `consensus_score`: Distribución de veredictos calculada en tiempo real.
- `synthesis`: Decision Brief generado por el orquestador.
- `history`: Lista de queries anteriores en la sesión.

---

### 6. ExpertChatSession
Historial de conversación 1:1 entre el usuario y un agente específico. En memoria.

**Atributos:**
- `conversation_id`: UUID de la conversación.
- `agent_id`: Agente con quien se conversa.
- `messages`: Lista de turnos `{ role: "user" | "assistant", content: string }`.
- `board_context`: Resumen del estado del board inyectado como contexto del agente.

---

## Reglas de Negocio

1. **Un board por URL**: La URL `/[boardId]` carga exclusivamente la configuración del board correspondiente. Si el board no existe, la app devuelve 404.

2. **Sin base de datos**: Todo el estado es en memoria. Las sesiones no sobreviven a un reinicio del servidor. Aceptable para el demo.

3. **Detección de idioma**: Los agentes responden en el mismo idioma de la pregunta. No hay configuración por board ni selector en la UI.

4. **Reconsideración dirigida**: El orquestador puede pedir a 1–3 agentes específicos que reconsideren su posición si detecta conflictos relevantes. No es una ronda global. Esta fase es opcional.

5. **Independencia de boards**: Los agentes de un board no comparten estado ni conocimiento con los de otro board.

6. **Consistencia de claims**: Si `consistency_check: true` en el board, cada respuesta pasa por una segunda llamada LLM que verifica claims sin respaldo. Si encuentra problemas, una tercera llamada los corrige manteniendo el veredicto.

7. **Configuración externa al código**: Los boards, agentes y system prompts se definen en archivos fuera del código (`boards/`). Cambiar un prompt o añadir un agente no requiere modificar ni desplegar la aplicación — solo actualizar los archivos y reiniciar.
