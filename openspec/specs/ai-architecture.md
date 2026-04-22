# Arquitectura AI — Advisory Board Simulator

Este documento describe la arquitectura de los agentes AI del proyecto. Es una app nueva, no una mejora del sistema anterior.

---

## Qué son los agentes aquí

No son agentes autónomos. Son **llamadas LLM estructuradas con personalidad, rol y formato de respuesta definidos externamente**. Cada agente representa un experto sintético que responde a una pregunta desde su perspectiva y produce un veredicto estructurado.

Para el demo, esto es suficiente. La arquitectura está diseñada para escalar a agentes autónomos reales sin reescribir nada.

---

## Capacidades

### Demo (implementación inicial)
- Responder como un rol/persona definida en config externa
- Output estructurado con veredicto y campos parseados
- Respuesta en el idioma de la pregunta (detección automática)
- Pseudo-comunicación: el orquestador decide qué agentes deben reconsiderar su posición en base a conflictos o relevancia entre respuestas

### Visión futura (no en el demo — la interfaz debe soportarlo)
- Debate real entre agentes (un agente responde a otro directamente)
- Uso de herramientas externas (búsqueda, cálculos, datos)
- Decisión autónoma sobre pasos siguientes
- Razonamiento multi-paso con loops

---

## Principio de diseño central

**Interfaz estable, implementación swappable.**

La interfaz de agente se define hoy y no cambia. La implementación interna puede evolucionar de llamadas directas a LangGraph sin tocar el board, el orquestador ni el frontend.

```
Board
  └── Agent (interfaz: recibe pregunta → devuelve AgentResponse)
        ├── demo: llamada LLM directa via LiteLLM
        └── futuro: LangGraph node / CrewAI agent
```

---

## Interfaz de agente

```python
class BaseAgent(ABC):
    @abstractmethod
    async def respond(
        self,
        question: str,
        context: AgentContext,
    ) -> AsyncGenerator[AgentChunk, None]:
        ...
```

### AgentContext
```python
@dataclass
class AgentContext:
    query_type: str                   # evaluation | landscape | advisory
    peer_summary: str | None          # resumen de posiciones del resto del board
    reconsider_reason: str | None     # por qué el orquestador pide reconsiderar
    board_context: str | None         # contexto general del board
```

### AgentChunk (streaming)
```python
@dataclass
class AgentChunk:
    agent_id: str
    text: str
    done: bool
    verdict: Verdict | None           # solo en el chunk final
```

### Verdict
```python
@dataclass
class Verdict:
    recommendation: str               # valor según query_type (ver prompt-architecture.md)
    primary_concern: str
    path_forward: str
    blocking_issue: str | None
    condition: str | None
    strength: str | None
    gap: str | None
```

---

## Orquestador

El orquestador coordina a los agentes. No sabe nada del interior de cada agente — solo llama `respond()` y consume el stream.

### Flujo

1. **Clasificación**: LLM rápido clasifica la pregunta como `evaluation`, `landscape` o `advisory`
2. **Ronda 1**: lanza todos los agentes del board en paralelo (con batching configurable para respetar rate limits)
3. **Análisis de conflictos**: identifica qué agentes tienen posiciones contradictorias o relevantes entre sí
4. **Ronda de reconsideración** (opcional): el orquestador selecciona 1–3 agentes y les pasa el contexto de por qué deben reconsiderar. No es una ronda global — es quirúrgica.
5. **Síntesis**: genera el Decision Brief con el Consensus Score y los puntos clave

### Pseudo-comunicación

El orquestador decide si dos o más agentes deben reconsiderar. No es debate directo entre agentes — es el orquestador que actúa como moderador. En la UI esto aparece como una fase visible: *"The orchestrator has asked Dr. X and Dr. Y to reconsider their position."*

### Configuración de concurrencia

```python
AGENT_BATCH_SIZE: int    # agentes por batch (default: 3)
AGENT_BATCH_DELAY: float # segundos entre batches (default: 0)
```

Configurable por variable de entorno según el tier de API disponible.

---

## Configuración de agentes

Los agentes se definen en archivos externos al código. Ver `app-architecture.md` para la estructura de carpetas.

Cada agente en `board.yaml`:
```yaml
agents:
  - id: cto
    name: Dr. Sarah Chen
    credentials: PhD Computer Science, MIT
    seat: Technology
    specialty: AI systems, scalability
    avatar: /avatars/cto.png
    prompt_file: agents/cto.md
```

El `prompt_file` es texto libre en markdown. Define: rol, mandato, rúbrica de evaluación, estilo de comunicación. Ver `prompt-architecture.md` para la estructura interna.

---

## RAG

Para el demo: TF-IDF (scikit-learn). Suficiente si los documentos de conocimiento están bien curados.

La interfaz se abstrae para migración futura a embeddings semánticos:

```python
class BaseRetriever(ABC):
    @abstractmethod
    def retrieve(self, query: str, n_results: int = 4) -> str:
        ...
```

Implementaciones:
- `TFIDFRetriever` — demo
- `SemanticRetriever` — futuro (sentence-transformers + ChromaDB o SQLite-vec)

Cada agente tiene su propio directorio de conocimiento: `boards/{board_id}/knowledge/{agent_id}/`.

---

## Frameworks

| Framework | Estado | Razón |
|-----------|--------|-------|
| Llamadas directas (LiteLLM) | ✅ Usar ahora | Simple, transparente, suficiente para el demo |
| LangGraph | Candidato futuro | Stateful, graph-based, modela debate como edges, tool use integrado, compatible con LiteLLM |
| CrewAI | Descartado por ahora | Menos control fino, muy opinionated |
| Anthropic SDK nativo | Descartado | Lock-in de proveedor |

**LangGraph** es el candidato para la migración futura porque:
- El debate entre agentes se modela naturalmente como edges en un grafo
- Estado persistente entre pasos
- Tool use integrado
- No hay lock-in de modelo (compatible con LiteLLM)

La interfaz `BaseAgent` está diseñada para que cada agente sea un nodo de LangGraph cuando llegue el momento.

---

## Idioma

Los agentes detectan el idioma de la pregunta y responden en ese idioma. Esto va en el system prompt base de todos los agentes:

> Respond in the same language the user used in their question.

No hay configuración por board ni selector en la UI.

---

## Modelo LLM

Configurable por variable de entorno, no hardcodeado:

```
LLM_MODEL=anthropic/claude-haiku-4-5-20251001   # default (demo)
LLM_MODEL=anthropic/claude-sonnet-4-6            # recomendado para producción
```

LiteLLM abstrae el proveedor. Cambiar de Claude a GPT-4o es cambiar una variable de entorno.
